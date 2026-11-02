# TARMOQ XAVFSIZLIGI — A6 AMALIY MASHG‘ULOTI QO‘LLANMASI
**1-VARIANT: WireGuard orqali ichki serverga eng kam ruxsatli masofaviy kirish (Least Privilege Remote Access via WireGuard)**

---

> [!IMPORTANT]
> **ISHLASH MUHITI VA MAXSUS SHARTLAR (DIQQAT BILAN O‘QING):**
> 1. **Lokal Wi-Fi Tarmog‘i:** Ushbu laboratoriya universitet tarmog‘ida emas, **shaxsiy / uy lokal Wi-Fi** tarmog‘ida bajariladi. Shu sababli universitet proksisi (`http://172.18.1.1:2500`) yoki universitet fayl serveri (`\\172.18.233.234`) **ishlatilmaydi**. Virtual mashinalar paketlarni yuklash va yangilashda VMware NAT orqali to‘g‘ridan-to‘g‘ri lokal internetingizdan foydalanadi (hech qanday `99proxy` yoki `export http_proxy` kerak emas).
> 2. **Asosiy Operatsion Tizim (Host OS):** Asosiy kompyuter — **Ubuntu Linux** (Windows EMAS!). Virtualizatsiya dasturi — **VMware Workstation Pro / Player for Linux**. Barcha host buyruqlari Linux Bash terminalida bajariladi (`sha256sum`, `ls`, `mkdir`, `cp`).
> 3. **Virtual Mashinalar (Guests):** VMware ichida **3 ta Ubuntu Server 24.04 LTS** virtual mashinasi ishga tushiriladi:
>    - `VPN-GW` — VPN Shlyuzi va nftables xavfsizlik devori (2 ta interfeys: WAN va LAN);
>    - `REMOTE-CLIENT` — Masofadagi xodim / mijoz qurilmasi (1 ta interfeys: WAN);
>    - `APP-SRV` — Faqat ichki tarmoqdan boshqariladigan himoyalangan server (1 ta interfeys: LAN).
> 4. **CLI Komandalar Tushuntirishi:** Qo‘llanmadagi har bir buyruq, uning barcha kalitlari (flaglari), argumentlari va nima vazifani bajarishi batafsil tushuntirilgan.

---

## MUNDARIJA
1. [1. Mavzuning Asosiy Maqsadi va Vazifalari](#1-mavzuning-asosiy-maqsadi-va-vazifalari)
2. [2. Muhitni Sozlash: Ubuntu Host, VMware va Lokal Wi-Fi](#2-muhitni-sozlash-ubuntu-host-vmware-va-lokal-wi-fi)
3. [3. Har Bir Mavzu Bo‘yicha Mukammal Nazariy Qo‘llanma](#3-har-bir-mavzu-boyicha-mukammal-nazariy-qollanma)
   - 3.1. WireGuard VPN protokoli, Noise Protocol Framework va Cryptokey Routing
   - 3.2. Split-Tunnel va Full-Tunnel arxitekturalari: Farqlari va Routing Loop xavfi
   - 3.3. nftables yordamida "Default Deny" va Eng Kam Ruxsatlar (Least Privilege) siyosati
   - 3.4. Ko‘p Qatlamli Himoya (Defense in Depth) va OpenSSH Hardening
   - 3.5. Paketlar darajasida tahlil: Outer (shifrlangan) va Inner (ochiq) trafik
4. [4. Tarmoq Topologiyasi, Adreslash Rejasi va Matritsa](#4-tarmoq-topologiyasi-adreslash-rejasi-va-matritsa)
5. [5. Mashqni Bajarishning Bosqichma-bosqich Yo‘riqnomasi](#5-mashqni-bajarishning-bosqichma-bosqich-yoriqnomasi)
   - 0-bosqich: Ubuntu hostda VMware muhitini va VM nusxalarini tayyorlash
   - 1-bosqich: Lokal internet (NAT) orqali paketlarni o‘rnatish va Snapshot olish
   - 2-bosqich: Laboratoriya adapterlarini ulash va bazaviy IP manzillarni biriktirish
   - 3-bosqich: T1 Testi — VPNsiz holatda APP-SRV ga ulanishning bloklanganini tekshirish (Baseline)
   - 4-bosqich: APP-SRV da Nginx va SSH xizmatlarini xavfsiz sozlash (Hardening)
   - 5-bosqich: WireGuard kriptografik kalitlarini xavfsiz yaratish va almashish
   - 6-bosqich: VPN-GW da WireGuard shlyuzini sozlash va ishga tushirish
   - 7-bosqich: REMOTE-CLIENT da split-tunnel mijozini sozlash va handshake tekshiruvi
   - 8-bosqich: VPN-GW da eng kam ruxsatli nftables xavfsizlik siyosatini joriy etish
   - 9-bosqich: Funksional va Salbiy Testlarni o‘tkazish (T1–T7)
   - 10-bosqich: Paket darajasidagi tahlil: tcpdump yordamida Outer va Inner PCAP olish
   - 11-bosqich: Konfiguratsiya auditi, hisobot paketini yig‘ish, SHA-256 va Rollback
6. [6. Har Bir CLI Buyruqning Mukammal Tahlili](#6-har-bir-cli-buyruqning-mukammal-tahlili)
7. [7. Natijalarni Qayd Etish Jadvallari va Hisobot Namunasi](#7-natijalarni-qayd-etish-jadvallari-va-hisobot-namunasi)
8. [8. O‘qituvchi Uchun Himoya Savollari va Mukammal Javoblar (14 ta savol)](#8-oqituvchi-uchun-himoya-savollari-va-mukammal-javoblar)
9. [9. Foydalanilgan Rasmiy Manbalar](#9-foydalanilgan-rasmiy-manbalar)

---

## 1. MAVZUNING ASOSIY MAQSADI VA VAZIFALARI

Ushbu amaliy mashg‘ulotning asosiy maqsadi — talabaga zamonaviy, tezkor va kriptografik jihatdan mustahkam **WireGuard VPN** protokoli asosida masofaviy xavfsiz ulanishni (Remote Access VPN) loyihalash, sozlash, xavfsizlik devori (**nftables**) orqali qat’iy filtrlash hamda ko‘p qatlamli himoya (**Defense in Depth**) tamoyillarini amalda joriy etish ko‘nikmalarini berishdir.

```
+----------------------------------------------------------------------------------------------------+
|                                    A6 XAVFSIZLIK ARXITEKTURASI                                     |
|                                                                                                    |
|   +-------------------+        Shifrlangan UDP/51820       +-------------------+                   |
|   |   REMOTE-CLIENT   | =================================> |      VPN-GW       |                   |
|   |  (Masofaviy Xodim)|        (WireGuard Tunnel)          | (Shlyuz + nftables|                   |
|   +-------------------+                                    +-------------------+                   |
|      172.30.60.10                                                |       |                         |
|      (wg0: 10.60.99.10)                                          |       |                         |
|             \                                                    |       |                         |
|              \--- Bevosita WAN->LAN ulanish urinishi ------------/       | Faqat ruxsat etilgan    |
|                   (nftables tomonidan DROP qilinadi!)                    | TCP: 22, 80 va ICMP     |
|                                                                          v                         |
|                                                                +-------------------+               |
|                                                                |      APP-SRV      |               |
|                                                                | (Ichki Veb/SSH)   |               |
|                                                                +-------------------+               |
|                                                                   10.60.10.20                      |
+----------------------------------------------------------------------------------------------------+
```

### Mashg‘ulotning Asosiy Vazifalari:
1. **Izolyatsiyalangan Tarmoq Muhitini Yaratish:** Real tarmoqdan va Internetdan to‘liq ajratilgan ikkita virtual segment (`A6-WAN` va `A6-LAN`) tashkil etish va 3 ta mustaqil Ubuntu Server virtual mashinasini joylashtirish.
2. **Kriptografik Kalitlar Xavfsizligi:** WireGuard va SSH uchun assimetrik kalitlar juftligini yaratish, maxfiy kalitlar (`private key`) xavfsizligini ta’minlash (`chmod 600`) hamda faqat ochiq kalitlar (`public key`) almashinuvini tashkil etish.
3. **Split-Tunnel Arxitekturasini Joriy Etish:** Mijoz tomonda barcha trafikni emas, faqat ichki korporativ tarmoq (`10.60.10.0/24`) va tunnel manzilini (`10.60.99.0/24`) VPN orqali yo‘naltirish, marshrutlash ilmoqlari (routing loops) xavfini bartaraf etish.
4. **Eng Kam Ruxsatlar Siyosati (Least Privilege via nftables):** VPN shlyuzida barcha tranzit va kiruvchi trafikni sukut bo‘yicha taqiqlash (`policy drop`), faqat shifrlangan WireGuard portiga (`UDP 51820`) va tunnel ichidan faqat APP-SRV ning belgilangan portlariga (`TCP 22, 80` va `ICMP echo`) ruxsat berish.
5. **Ko‘p Qatlamli Himoya (Defense in Depth):** Ichki serverda SSH xizmatini mustahkamlash (parol bilan kirishni to‘liq o‘chirish, root foydalanuvchini bloklash, faqat Ed25519 ochiq kaliti bilan kirishni yoqish).
6. **Paket Darajasidagi Tahlil va Korrelyatsiya:** `tcpdump` yordamida tashqi segmentdagi shifrlangan UDP/51820 oqimini (`A6_outer.pcap`) va ichki segmentdagi dekapsulyatsiyalangan TCP/22, TCP/80 oqimini (`A6_inner.pcap`) ushlash, ularni taqqoslash va kriptografik dalillarni isbotlash.
7. **Tizim Auditi va Rollback:** Barcha konfiguratsiyalarni xavfsiz eksport qilish, SHA-256 nazorat summalarini hisoblash va tizimni xavfsiz holatga qaytarish.

---

## 2. MUHITNI SOZLASH: UBUNTU HOST, VMWARE VA LOKAL WI-FI

Laboratoriya ishi talabaning shaxsiy kompyuterida — **Ubuntu Linux** operatsion tizimi ostida bajariladi.

### Muhit parametrlari:
- **Host tizim:** Ubuntu 22.04 LTS yoki 24.04 LTS (Desktop/Laptop).
- **Virtualizatsiya dasturi:** VMware Workstation Pro yoki VMware Player for Linux (`vmware --version`).
- **Internet ulanishi:** Shaxsiy uy / mobil lokal Wi-Fi (universitet proksisi va cheklovlarisiz).
- **Virtual mashinalar:** 3 ta Ubuntu Server 24.04 LTS nusxasi.

```
+------------------------------------------------------------------------------------+
|                         UBUNTU HOST (Lokal Wi-Fi tarmog'i)                         |
|                                                                                    |
|   +----------------------------------------------------------------------------+   |
|   |                     VMware Workstation Pro / Player                        |   |
|   |                                                                            |   |
|   |   +-------------------+   +--------------------+   +-------------------+   |   |
|   |   |   REMOTE-CLIENT   |   |       VPN-GW       |   |      APP-SRV      |   |   |
|   |   |  (Ubuntu Server)  |   |  (Ubuntu Server)   |   |  (Ubuntu Server)  |   |   |
|   |   +---------+---------+   +-----+--------+-----+   +---------+---------+   |   |
|   |             |                   |        |                   |             |   |
|   |             |   A6-WAN          |        |   A6-LAN          |             |   |
|   |             +===================+        +===================+             |   |
|   |                 LAN Segment                  LAN Segment                   |   |
|   |             (172.30.60.0/24)             (10.60.10.0/24)                   |   |
|   +----------------------------------------------------------------------------+   |
+------------------------------------------------------------------------------------+
```

### VMware da LAN Segmentlarni yaratish:
1. VMware oynasida istalgan VM sozlamalariga kiring (`VM -> Settings`).
2. `Network Adapter` bo‘limini tanlang.
3. `LAN Segment` tugmasini bosing va ochilgan oynada ikkita segment qo‘shing:
   - `A6-WAN`
   - `A6-LAN`
4. Bu segmentlar to‘liq izolyatsiyalangan bo‘lib, ularda DHCP server bo‘lmaydi. Barcha IP manzillar qo‘lda statik biriktiriladi.

---

## 3. HAR BIR MAVZU BO‘YICHA MUKAMMAL NAZARIY QO‘LLANMA

### 3.1. WireGuard VPN Protokoli, Noise Protocol Framework va Cryptokey Routing

**WireGuard** — bu zamonaviy, nihoyatda ixcham (atigi ~4000 qator C kodidan iborat bo‘lib, Linux yadrosi ichida ishlaydi), yuqori tezlikka va ilg‘or kriptografiyaga ega bo‘lgan VPN protokol hisoblanadi. U OpenVPN yoki IPsec kabi eski, murakkab va sekin protokollardan tubdan farq qiladi.

#### Asosiy Kriptografik Prinsiplari:
- **Noise Protocol Framework:** WireGuard Noise IK handshake mexanizmidan foydalanadi. U 1 ta RTT (Round Trip Time) ichida o‘zaro autentifikatsiya va sessiya kalitlarini generatsiya qiladi.
- **Curve25519 (ECDH):** Elliptik egri chiziqlar asosidagi Diffie-Hellman algoritmi orqali maxfiy kalitlar almashiladi. Kalit uzunligi atigi 32 bayt (256 bit).
- **ChaCha20 va Poly1305 (AEAD):** ChaCha20 simmetrik oqimli shifrlash algoritmi, Poly1305 esa ma’lumotlarning butunligini va haqiqiyligini tekshiruvchi MAC (Message Authentication Code) hisoblanadi.
- **BLAKE2s:** SHA-256 va MD5 dan ancha tez va xavfsiz bo‘lgan kriptografik xeshlash funksiyasi.
- **SipHash24:** Hash-table to‘qnashuvlariga (hash collision DoS) qarshi himoya.

```
+-----------------------------------------------------------------------------+
|                     WIREGUARD CRYPTOKEY ROUTING MODELI                      |
|                                                                             |
|   Yadro Routing Jadvali                WireGuard Cryptokey Routing Jadvali  |
|   +----------------------+            +----------------------------------+  |
|   | 10.60.10.0/24 -> wg0 | ---------> | Peer: <GW_Public_Key>            |  |
|   +----------------------+            |   AllowedIPs: 10.60.10.0/24      |  |
|                                       |   Endpoint: 172.30.60.1:51820    |  |
|                                       +----------------------------------+  |
|                                                        |                    |
|                                                        v                    |
|                                       Curve25519 + ChaCha20-Poly1305        |
|                                       Shifrlangan UDP/51820 paketi hosil    |
|                                       bo'lib, tashqi tarmoqqa yuboriladi.   |
+-----------------------------------------------------------------------------+
```

#### Cryptokey Routing Tushunchasi:
WireGuardning eng asosiy innovatsiyasi — bu **Cryptokey Routing** (Kripto-kalitli marshrutlash) dir.
Har bir virtual tarmoq interfeysi (`wg0`) o‘zining maxfiy kalitiga va bog‘langan bir nechta peerlar (hamkorlar) ro‘yxatiga ega bo‘ladi.
Har bir peer uchun ikkita asosiy parametr belgilanadi:
1. `PublicKey` — peerni autentifikatsiya qiluvchi ochiq kalit.
2. `AllowedIPs` — ushbu peerga tegishli bo‘lgan IP manzillar yoki quyi tarmoqlar ro‘yxati.

**Cryptokey Routing qanday ishlaydi?**
- **Paket jo‘natilayotganda (Outbound):** Agar Linux yadrosi paketni `wg0` interfeysiga yo‘naltirsa, WireGuard paketning manziliga (`Destination IP`) qaraydi va `AllowedIPs` ro‘yxatida shu manzil ko‘rsatilgan Peerni topadi. Paket ushbu peerning ochiq kaliti bilan shifrlanadi va uning `Endpoint` manziliga (UDP orqali) jo‘natiladi.
- **Paket qabul qilinayotganda (Inbound):** Tashqi interfeysdan UDP paketi kelib tushganda, WireGuard uni qaysi peer jo‘natganini aniqlaydi va shifrni yechadi. Shifrdan chiqqan paketning jo‘natuvchi manzili (`Source IP`) ushbu peerning `AllowedIPs` ro‘yxatida bormi-yo‘qligini tekshiradi! Agar `Source IP` ro‘yxatda bo‘lmasa, paket **zudlik bilan tashlab yuboriladi (drop qilinadi)**. Bu IP-spoofing hujumlarining oldini avtomatik oladi.

#### Stealth Mode (Jimgina rejim):
WireGuard serveri o‘z portini (masalan, UDP 51820) nmap yoki port-skanerlarga bildirmaydi. Agar kelgan UDP paket to‘g‘ri kriptografik kalit bilan imzolanmagan bo‘lsa, WireGuard unga hech qanday javob (hatto ICMP Port Unreachable ham) qaytarmaydi. Tashqi kuzatuvchi uchun port butunlay yopiqdek tuyuladi.

---

### 3.2. Split-Tunnel va Full-Tunnel Arxitekturalari: Farqlari va Routing Loop Xavfi

Masofaviy VPN ulanishlarda ikki xil marshrutlash modeli qo‘llaniladi:

```
               SPLIT-TUNNEL                                   FULL-TUNNEL
     +-------------------------------+              +-------------------------------+
     |         REMOTE-CLIENT         |              |         REMOTE-CLIENT         |
     +---------------+---------------+              +---------------+---------------+
            /                 \                                     |
           /                   \                                    |
(Ichki LAN trafigi)    (Oddiy Internet trafigi)             (BARCHA TRAFIK: 0.0.0.0/0)
         /                       \                                  |
        v                         v                                 v
   [ wg0 Tunnel ]          [ enp0s8 WAN ]                    [ wg0 Tunnel ]
        |                         |                                 |
        v                         v                                 v
     APP-SRV                   Internet                          VPN-GW
  (10.60.10.20)             (Masalan, YouTube)                      |
                                                                    v
                                                            (Hamma trafik shlyuz
                                                             orqali aylanadi)
```

1. **Full-Tunnel (`AllowedIPs = 0.0.0.0/0`):**
   - Mijozning barcha internet va lokal trafigi to‘liq VPN tunnel orqali shlyuzga yuboriladi.
   - *Afzalligi:* To‘liq nazorat va xavfsizlik.
   - *Kamchiligi:* VPN shlyuziga ulkan yuklama tushadi, xodimning shaxsiy internet tezligi pasayadi.
2. **Split-Tunnel (`AllowedIPs = 10.60.99.0/24, 10.60.10.0/24`):**
   - Faqat korporativ ichki resurslarga (`10.60.10.0/24`) tegishli trafik tunnelga yo‘naltiriladi. Tashqi internet trafigi mijozning o‘z lokal tarmog‘i orqali ketadi.
   - *Afzalligi:* Shlyuz trafigi tejaladi, xodimning interneti sekinlashmaydi.
   - *Talab qilinadigan ehtiyotkorlik:* Endpoint marshruti xatosi!

#### Routing Loop (Marshrutlash ilmog‘i) Xavfi:
Agar `REMOTE-CLIENT` da `AllowedIPs` ro‘yxatiga xato qilib VPN-GW ning tashqi WAN IP manzili (`172.30.60.1`) qo‘shib qo‘yilsa yoki `AllowedIPs = 0.0.0.0/0` bo‘lib, `wg-quick` tashqi endpoint uchun alohida istisno marshrut qo‘shmasa:
- WireGuard shifrlangan UDP paketni `172.30.60.1:51820` ga jo‘natmoqchi bo‘ladi;
- Lekin yadro `172.30.60.1` ga borish uchun paketni yana `wg0` ga tiqadi;
- Paket o‘z-o‘zini qayta shifrlashga tushib, cheksiz aylanma sikl (**Routing Loop**) hosil bo‘ladi va butun tarmoq aloqasi bir soniyada uziladi!
Shu sababli, split-tunnelda `AllowedIPs` qat’iy tarzda faqat ichki manzillar bilan cheklanadi.

---

### 3.3. nftables Yordamida "Default Deny" va Eng Kam Ruxsatlar (Least Privilege) Siyosati

VPN tunnelining o‘rnatilishi xavfsizlikning faqat birinchi qadamidir. VPN o‘zi kifoya qilmaydi. Agar VPN shlyuzida xavfsizlik devori (firewall) bo‘lmasa, masofadagi kompyuterga tushgan virus yoki buzg‘unchi butun ichki tarmoqdagi barcha server va printerlarga erkin hujum qila oladi.

Shu sababli, **Eng Kam Ruxsatlar Tamoyili (Principle of Least Privilege)** asosida `nftables` xavfsizlik devori o‘rnatiladi:

```
+---------------------------------------------------------------------------------------+
|                              VPN-GW nftables SIYOSATI                                 |
|                                                                                       |
|   [ INPUT zanjiri: policy DROP ]                                                      |
|   ├── Loopback (lo) -----------------------------------------------------> ACCEPT     |
|   ├── Allaqachon o'rnatilgan sessiyalar (ct state established,related) -> ACCEPT     |
|   ├── A6-WAN dan WireGuard porti (enp0s8, udp dport 51820) --------------> ACCEPT     |
|   ├── A6-WAN dan Gateway pingi (enp0s8, icmp type echo-request) ---------> ACCEPT     |
|   ├── wg0 tunnelidan Gateway pingi (wg0, 10.60.99.10, echo-request) -----> ACCEPT     |
|   └── Qolgan har qanday kiruvchi paket ----------------------------------> DROP       |
|                                                                                       |
|   [ FORWARD zanjiri: policy DROP ]                                                    |
|   ├── Javob paketlari (ct state established,related) --------------------> ACCEPT     |
|   ├── BEVOSITA WAN->LAN OQIMI (enp0s8 -> enp0s9, daddr 10.60.10.20) -----> DROP (LOG) |
|   ├── Tunneldan APP-SRV ga SSH va HTTP (wg0 -> enp0s9, dport {22, 80}) --> ACCEPT     |
|   ├── Tunneldan APP-SRV ga ICMP ping (wg0 -> enp0s9, echo-request) ------> ACCEPT     |
|   └── Boshqa barcha tranzit paketlar ------------------------------------> DROP       |
+---------------------------------------------------------------------------------------+
```

#### Xavfsizlik Qoidalarining Ahamiyati:
1. **Direct Bypass Drop:** Agar kimdir VPN tunnelni yoqmasdan, shunchaki VPN-GW orqali `10.60.10.20` ga bevosita paket yuborishga urinsa (`enp0s8 -> enp0s9`), nftables bu paketni maxsus hisoblagich (**counter**) bilan `DROP` qiladi. Bu qoida T7 testida tekshiriladi!
2. **Port Cheklovi:** Masofaviy mijoz hatto tunnel ichida bo‘lsa ham, APP-SRV ning faqat 22 (SSH) va 80 (HTTP) portlariga kira oladi. Serverdagi boshqa hech bir port ochilmaydi.

---

### 3.4. Ko‘p Qatlamli Himoya (Defense in Depth) va OpenSSH Hardening

**Defense in Depth (Chuqurlashtirilgan / Ko‘p Qatlamli Himoya):** Xavfsizlik birgina to‘siqqa (masalan, faqat VPNga) tayanib qolmasligi kerak.
- 1-qatlam: WireGuard tunnel shifrlash va peer autentifikatsiyasi (Noise Protocol).
- 2-qatlam: VPN-GW da nftables tarmoq darajasidagi filtrlash siyosati.
- 3-qatlam: Ichki serverning o‘zida dastur darajasidagi xavfsizlik — **OpenSSH Hardening**.

```
              1-QATLAM                     2-QATLAM                     3-QATLAM
        [ WireGuard Kaliti ]         [ nftables Qoidalari ]       [ SSH Ed25519 Kaliti ]
                 |                            |                            |
                 v                            v                            v
         Qurilma tarmoqqa             Faqat 22 va 80-port           Faqat 'remoteops'
         ulanish huquqiga             ochiq, qolgan hamma           parolsiz, faqat
         ega ekanligini               narsa bloklangan              Ed25519 kaliti bilan
         tasdiqlaydi.                 (least privilege).            tizimga kira oladi.
```

#### SSH ni Kuchaytirish (Hardening) Sozlamalari:
- `PermitRootLogin no`: Tizim administratori (root) nomidan masofadan kirish mutlaqo man etiladi.
- `PasswordAuthentication no`: Parol orqali kirish o‘chiriladi. Hech qanday brute-force (parol tanlash) hujumi ish bermaydi.
- `PubkeyAuthentication yes`: Faqat kriptografik ochiq kalit (Ed25519) bilan kirishga ruxsat beriladi.
- `AllowUsers remoteops`: Faqat maxsus yaratilgan `remoteops` nomli cheklangan foydalanuvchigina SSH orqali kira oladi.
- `X11Forwarding no` va `AllowTcpForwarding no`: SSH tunnel orqali boshqa tarmoq portlarini aylanib o‘tish yoki grafik interfeysni masofadan chiqarish taqiqlanadi.

---

### 3.5. Paketlar Darajasida Tahlil: Outer (Shifrlangan) va Inner (Ochiq) Trafik

WireGuard ma’lumotlarni kapsulyatsiya (encapsulation) qiladi. Laboratoriyaning eng muhim tahliliy qismi — bu tashqi va ichki tarmoqdagi paketlarni ushlab, solishtirishdir:

```
+--------------------------------------------------------------------------------------------------+
|                            PAKETLARNING TAQQOSLANISHI (PCAP TAHLILI)                             |
|                                                                                                  |
|   1. TASHQI TRAFIK (A6_outer.pcap - enp0s8 interfeysida):                                        |
|   +-------------------+-------------------+--------------------------------------------------+   |
|   | IP Manba:         | IP Maqsad:        | Protokol va Ma'lumot:                            |   |
|   | 172.30.60.10      | 172.30.60.1       | UDP Port: 51820 (WireGuard Data - TO'LIQ SHIFR)  |   |
|   +-------------------+-------------------+--------------------------------------------------+   |
|   * Tashqi kuzatuvchi kim kim bilan bog'lanayotganini ko'radi, ammo ichida nima borligini        |
|     (HTTP sahifa matni, SSH buyruqlar, parollar) aslo ko'ra olmaydi!                             |
|                                                                                                  |
|   2. ICHKI TRAFIK (A6_inner.pcap - enp0s9 interfeysida):                                         |
|   +-------------------+-------------------+--------------------------------------------------+   |
|   | IP Manba:         | IP Maqsad:        | Protokol va Ma'lumot:                            |   |
|   | 10.60.99.10       | 10.60.10.20       | TCP Port: 80 (HTTP GET / HTTP 200 OK HTML matni) |   |
|   | 10.60.99.10       | 10.60.10.20       | TCP Port: 22 (SSHv2 Protokol sessiyasi)          |   |
|   +-------------------+-------------------+--------------------------------------------------+   |
|   * Shlyuz shifrni yechgach, ichki tarmoqqa sof IP paketlarni chiqaradi.                         |
+--------------------------------------------------------------------------------------------------+
```

---

## 4. TARMOQ TOPOLOGIYASI, ADRESLASH REJASI VA MATRITSA

### 4.1. Virtual Mashinalar va Resurslar Jadvali

| VM Nomi | Vazifasi / Roli | vCPU | RAM | Disk | VMware Adapter 1 | VMware Adapter 2 |
| :--- | :--- | :---: | :---: | :---: | :--- | :--- |
| **VPN-GW** | WireGuard VPN shlyuzi va nftables firewall | 2 | 4 GB | 20 GB+ | `enp0s8` (LAN Segment: `A6-WAN`) | `enp0s9` (LAN Segment: `A6-LAN`) |
| **REMOTE-CLIENT** | Masofadagi xodim / sinov mijozi | 2 | 2 GB | 20 GB+ | `enp0s8` (LAN Segment: `A6-WAN`) | — |
| **APP-SRV** | Ichki himoyalangan veb va SSH serveri | 2 | 2 GB | 20 GB+ | `enp0s8` (LAN Segment: `A6-LAN`) | — |

> [!NOTE]
> Interfeys nomlari (`enp0s8`, `enp0s9`) VM konfiguratsiyasiga qarab `ens33`, `ens37` yoki `eth0` bo‘lishi mumkin. Har doim `ip -br link` buyrug‘i orqali o‘z interfeys nomlaringizni aniqlab oling.

### 4.2. IP Adreslash Rejasi

| Segment / Interfeys | Tarmoq Prefiksi | Qurilma | IP Manzili | Izoh |
| :--- | :--- | :--- | :--- | :--- |
| **A6-WAN (Tashqi)** | `172.30.60.0/24` | VPN-GW (`enp0s8`) | `172.30.60.1` | VPN Endpoint IP manzili |
| | | REMOTE-CLIENT (`enp0s8`) | `172.30.60.10` | Mijozning tashqi IP manzili |
| **A6-LAN (Ichki)** | `10.60.10.0/24` | VPN-GW (`enp0s9`) | `10.60.10.1` | Ichki tarmoq Gateway manzili |
| | | APP-SRV (`enp0s8`) | `10.60.10.20` | Serverning ichki IP manzili |
| **WireGuard Overlay** | `10.60.99.0/24` | VPN-GW (`wg0`) | `10.60.99.1/24` | Shlyuzning tunneldagi manzili |
| | | REMOTE-CLIENT (`wg0`) | `10.60.99.10/32` | Mijozning tunneldagi manzili |

### 4.3. Tarmoq Kirish Huquqlari Matritsasi

| Qayerdan | Qayerga | Protokol / Port | nftables Qarori | Sabab va Izoh |
| :--- | :--- | :--- | :---: | :--- |
| `REMOTE-CLIENT` (WAN) | `VPN-GW` (WAN) | UDP 51820 | **ACCEPT** | WireGuard shifrlangan tunnel ulanishi |
| `REMOTE-CLIENT` (WAN) | `VPN-GW` (WAN) | ICMP (Ping) | **ACCEPT** | Tashqi aloqani diagnostika qilish |
| `REMOTE-CLIENT` (WAN) | `APP-SRV` (LAN) | Barcha portlar | **DROP** | **Bevosita WAN->LAN chetlab o‘tish taqiqlanadi!** |
| `REMOTE-CLIENT` (wg0) | `VPN-GW` (wg0) | ICMP (Ping) | **ACCEPT** | Tunnel ishlashini diagnostika qilish |
| `REMOTE-CLIENT` (wg0) | `APP-SRV` (LAN) | TCP 80 (HTTP) | **ACCEPT** | Ichki veb-xizmatga kirish |
| `REMOTE-CLIENT` (wg0) | `APP-SRV` (LAN) | TCP 22 (SSH) | **ACCEPT** | Faqat `remoteops` kaliti bilan kirish |
| `REMOTE-CLIENT` (wg0) | `APP-SRV` (LAN) | ICMP (Ping) | **ACCEPT** | Ichki server bilan aloqani tekshirish |
| `REMOTE-CLIENT` (wg0) | Boshqa manzillar | Barcha portlar | **DROP** | Minimal ruxsat tamoyili |

---

## 5. MASHQNI BAJARISHNING BOSQICHMA-BOSQICH YO‘RIQNOMASI

### 0-BOSQICH: Ubuntu Hostda VMware Muhitini va VM Nusxalarini Tayyorlash

1. **Ubuntu Host tizimingizda** laboratoriya uchun alohida katalog yarating:
   ```bash
   mkdir -p ~/A6-LAB/{VPN-GW,REMOTE-CLIENT,APP-SRV,Report}
   ```
2. Ubuntu Server 24.04 bazaviy obrazini (yoki ISO orqali yaratilgan shablonni) yuqoridagi 3 ta alohida katalogga nusxalang (`cp -r ...`).
3. VMware Workstation Linux dasturini oching (`vmware &`).
4. `File -> Open` orqali har bir katalogdagi `.vmx` faylni alohida oching.
5. Mashinalarni birinchi marta yoqqanda VMware "Did you move or copy it?" deb so‘rasa, albatta **"I Copied It"** tugmasini bosing (bu har bir VM ga yangi unikal MAC manzil beradi).
6. Har bir VM konsoliga kirib, uning nomini (`hostname`) belgilang:
   - **VPN-GW mashinasida:**
     ```bash
     sudo hostnamectl set-hostname VPN-GW
     ```
   - **REMOTE-CLIENT mashinasida:**
     ```bash
     sudo hostnamectl set-hostname REMOTE-CLIENT
     ```
   - **APP-SRV mashinasida:**
     ```bash
     sudo hostnamectl set-hostname APP-SRV
     ```
   - Terminaldan chiqib qayta kiring (`exit` yoki yangi shell oching).

---

### 1-BOSQICH: Lokal Internet (NAT) Orqali Paketlarni O‘rnatish va Snapshot Olish

> [!IMPORTANT]
> **Lokal Wi-Fi Sharoiti:** Hozir har bir VM ning 1-tarmoq kartasi VMware NAT rejimida turgan bo‘lishi kerak. Virtual mashinalar to‘g‘ridan-to‘g‘ri sizning uyingizdagi Wi-Fi orqali internetga chiqadi. Hech qanday proksi (`172.18.1.1:2500`) sozlamang!

#### 1. VPN-GW va REMOTE-CLIENT mashinalarida zarur paketlarni o‘rnatish:
Ushbu ikki VMda quyidagi buyruqni bajaring:
```bash
sudo apt update && sudo apt install -y wireguard wireguard-tools nftables tcpdump iproute2 curl iputils-ping
```

#### 2. APP-SRV mashinasida zarur paketlarni o‘rnatish:
```bash
sudo apt update && sudo apt install -y openssh-server nginx curl iputils-ping iproute2
sudo systemctl enable --now ssh nginx
```

#### 3. O‘rnatilgan paketlarni tekshirish:
```bash
wg --version
nft --version
ssh -V
nginx -v
```

#### 4. NAT adapterini uzish va Bazaviy Snapshot (`A6-base`) olish:
1. Paketlar muvaffaqiyatli o‘rnatilgach, har uchala VMni o‘chiring:
   ```bash
   sudo poweroff
   ```
2. VMware sozlamalarida (`VM -> Settings`):
   - NAT adapterining `Connected` va `Connect at power on` belgilarini olib tashlang.
   - Har bir VM ni o‘zining laboratoriya segmentiga ulang (keyingi bosqichda ko‘rsatilgan).
3. VMware menyusidan har bir VM uchun toza holat snapshotini oling:
   - `VM -> Snapshot -> Take Snapshot...`
   - Nomi: **`A6-base`**
   - Izoh: *Paketlar o'rnatilgan, NAT uzilgan, tarmoq sozlanishga tayyor.*

---

### 2-BOSQICH: Laboratoriya Adapterlarini Ulash va Bazaviy IP Manzillarni Biriktirish

VMware sozlamalarida virtual adapterlarni quyidagicha ulang:
- **VPN-GW:**
  - 1-adapter: `LAN Segment` -> tanlang: **`A6-WAN`**
  - 2-adapter: `LAN Segment` -> tanlang: **`A6-LAN`**
- **REMOTE-CLIENT:**
  - 1-adapter: `LAN Segment` -> tanlang: **`A6-WAN`**
- **APP-SRV:**
  - 1-adapter: `LAN Segment` -> tanlang: **`A6-LAN`**

Mashinalarni yoqing. Har bir VM konsolida interfeys nomlarini tekshirib oling:
```bash
ip -br link
```
*(Quyidagi qo‘llanmada 1-adapter `enp0s8`, 2-adapter esa `enp0s9` deb yozilgan. Agar sizda `ens33`, `ens37` bo‘lsa, nomlarni moslang).*

#### 1. VPN-GW da IP manzillarni biriktirish va Routingni yoqish:
```bash
# Tashqi A6-WAN interfeysi:
sudo ip addr flush dev enp0s8
sudo ip addr add 172.30.60.1/24 dev enp0s8
sudo ip link set enp0s8 up

# Ichki A6-LAN interfeysi:
sudo ip addr flush dev enp0s9
sudo ip addr add 10.60.10.1/24 dev enp0s9
sudo ip link set enp0s9 up

# Linux yadrosida paketlarni marshrutlashni (IP Forwarding) yoqish:
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/60-a6-vpn.conf
```

#### 2. REMOTE-CLIENT da IP manzilni biriktirish:
```bash
sudo ip addr flush dev enp0s8
sudo ip addr add 172.30.60.10/24 dev enp0s8
sudo ip link set enp0s8 up
```

#### 3. APP-SRV da IP manzil va Gatewayni biriktirish:
```bash
sudo ip addr flush dev enp0s8
sudo ip addr add 10.60.10.20/24 dev enp0s8
sudo ip link set enp0s8 up

# Shlyuz sifatida VPN-GW ning ichki interfeysini ko'rsatish:
sudo ip route replace default via 10.60.10.1 dev enp0s8
```

---

### 3-BOSQICH: T1 Testi — VPNsiz Holatda APP-SRV ga Ulanishning Bloklanganini Tekshirish (Baseline)

Hozir VPN o‘rnatilmagan. Masofaviy mijozdan ichki serverga ulanish imkoni yo‘qligini tekshirib, bazaviy dalil (Baseline evidence) olamiz.

**REMOTE-CLIENT konsolida bajaring:**
```bash
# 1. VPN-GW ning tashqi WAN manzilini tekshirish (Ishlashi kerak):
ping -c 2 172.30.60.1

# 2. Ichki serverga marshrut borligini tekshirish (Yo'q bo'lishi kerak):
ip route get 10.60.10.20 || true

# 3. Ichki veb-serverga ulanishga urinish (Timeout yoki No route bo'lishi kerak):
curl --noproxy "*" --connect-timeout 3 http://10.60.10.20/ || true
```

> [!NOTE]
> **T1 Natijasi:** `172.30.60.1` ga ping boradi, ammo `10.60.10.20` ga ulanish mutlaqo amalga oshmaydi (`Failed to connect` yoki `Network is unreachable`). Bu to‘g‘ri bazaviy holatdir!

---

### 4-BOSQICH: APP-SRV da Nginx va SSH Xizmatlarini Xavfsiz Sozlash (Hardening)

#### 1. APP-SRV konsolida nazorat veb-sahifasini va maxsus foydalanuvchini yaratish:
```bash
# Sinov HTML sahifasini yozish:
echo 'A6 INTERNAL SERVICE' | sudo tee /var/www/html/index.html

# remoteops nomli alohida foydalanuvchini parolsiz yaratish:
sudo adduser --disabled-password --gecos '' remoteops

# Veb va SSH xizmatlari ishlab turganini tekshirish:
sudo systemctl enable --now nginx ssh
curl --noproxy "*" -s http://127.0.0.1/
```

#### 2. REMOTE-CLIENT konsolida SSH Ed25519 kalit juftligini yaratish:
```bash
# .ssh katalogini yaratish:
install -d -m 700 ~/.ssh

# Ed25519 kalit juftini parolsiz (-N '') yaratish:
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/a6_remoteops -N ''

# Ochiq kalitning fingerprintini ko'rish:
ssh-keygen -lf ~/.ssh/a6_remoteops.pub

# Ochiq kalit matnini ekranga chiqarish:
cat ~/.ssh/a6_remoteops.pub
```
*(Ekranga chiqqan `ssh-ed25519 AAAAC3... remoteops` qatorini to‘liq nusxalab oling).*

#### 3. APP-SRV konsoliga REMOTE-CLIENT ning ochiq kalitini joylashtirish:
```bash
sudo install -d -m 700 -o remoteops -g remoteops /home/remoteops/.ssh

# Quyidagi buyruqdagi <OCHIQ_KALIT_MATNI> o'rniga REMOTE-CLIENT da chiqqan kalitni qo'ying:
echo '<OCHIQ_KALIT_MATNI>' | sudo tee /home/remoteops/.ssh/authorized_keys

sudo chown remoteops:remoteops /home/remoteops/.ssh/authorized_keys
sudo chmod 600 /home/remoteops/.ssh/authorized_keys
```

#### 4. APP-SRV da SSHD xavfsizlik siyosatini (Hardening) joriy etish:
```bash
sudo tee /etc/ssh/sshd_config.d/00-a6-remote.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowUsers remoteops
MaxAuthTries 3
X11Forwarding no
AllowTcpForwarding no
EOF

# Konfiguratsiya sintaksisini tekshirish (xato chiqmasligi shart):
sudo sshd -t

# SSH xizmatini qayta yuklash:
sudo systemctl reload ssh

# Amaldagi samarali sozlamalarni tekshirish:
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers|maxauthtries'
```

---

### 5-BOSQICH: WireGuard Kriptografik Kalitlarini Xavfsiz Yaratish va Almashish

> [!CAUTION]
> Maxfiy kalitlar (`*private.key`) fayl ruxsati faqat `600` bo‘lishi shart va ular hech qachon o‘z mashinasini tark etmasligi kerak! Mashinalar o‘rtasida faqat va faqat **Public kalitlar** almashiladi.

#### 1. VPN-GW da kalit juftligini yaratish:
```bash
# Katalog xavfsizligini ta'minlash:
sudo install -d -m 700 /etc/wireguard

# Maxfiy kalitni yaratish va darhol 600 huquqini berish:
wg genkey | sudo tee /etc/wireguard/gw_private.key >/dev/null
sudo chmod 600 /etc/wireguard/gw_private.key

# Maxfiy kalit asosida ochiq kalitni generatsiya qilish:
sudo cat /etc/wireguard/gw_private.key | wg pubkey | sudo tee /etc/wireguard/gw_public.key >/dev/null
sudo chmod 644 /etc/wireguard/gw_public.key

# VPN-GW ning ochiq kalitini ko'rish:
cat /etc/wireguard/gw_public.key
```
*(Ushbu ochiq kalitni eslab qoling yoki nusxalang — bu `GW_PUBLIC_KEY`).*

#### 2. REMOTE-CLIENT da kalit juftligini yaratish:
```bash
sudo install -d -m 700 /etc/wireguard

wg genkey | sudo tee /etc/wireguard/client_private.key >/dev/null
sudo chmod 600 /etc/wireguard/client_private.key

sudo cat /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key >/dev/null
sudo chmod 644 /etc/wireguard/client_public.key

# REMOTE-CLIENT ning ochiq kalitini ko'rish:
cat /etc/wireguard/client_public.key
```
*(Ushbu ochiq kalitni eslab qoling yoki nusxalang — bu `CLIENT_PUBLIC_KEY`).*

---

### 6-BOSQICH: VPN-GW da WireGuard Shlyuzini Sozlash va Ishga Tushirish

**VPN-GW konsolida bajaring:**
1. REMOTE-CLIENT dan olingan ochiq kalitni o‘zgaruvchiga kiriting va konfiguratsiyani yarating:
```bash
# REMOTE-CLIENT ning ochiq kalitini so'raydi, kiritib Enter bosing:
read -r -p 'REMOTE-CLIENT public key: ' CLIENT_PUBLIC_KEY

GW_PRIVATE_KEY=$(sudo cat /etc/wireguard/gw_private.key)

sudo tee /etc/wireguard/wg0.conf >/dev/null <<EOF
[Interface]
Address = 10.60.99.1/24
ListenPort = 51820
PrivateKey = $GW_PRIVATE_KEY

[Peer]
PublicKey = $CLIENT_PUBLIC_KEY
AllowedIPs = 10.60.99.10/32
EOF

# Xotiradagi o'zgaruvchilarni tozalash:
unset GW_PRIVATE_KEY CLIENT_PUBLIC_KEY

# Fayl xavfsizligini ta'minlash:
sudo chmod 600 /etc/wireguard/wg0.conf

# Sintaksisni tekshirish:
sudo wg-quick strip wg0 >/dev/null

# WireGuard xizmatini ishga tushirish va tizim yuklanishiga qo'shish:
sudo systemctl enable --now wg-quick@wg0

# Holatni tekshirish:
sudo wg show wg0
ip -br addr show wg0
sudo ss -lunp | grep ':51820'
```

---

### 7-BOSQICH: REMOTE-CLIENT da Split-Tunnel Mijozini Sozlash va Handshake Tekshiruvi

**REMOTE-CLIENT konsolida bajaring:**
1. VPN-GW dan olingan ochiq kalitni o‘zgaruvchiga kiriting va konfiguratsiyani yarating:
```bash
# VPN-GW ning ochiq kalitini so'raydi, kiritib Enter bosing:
read -r -p 'VPN-GW public key: ' GW_PUBLIC_KEY

CLIENT_PRIVATE_KEY=$(sudo cat /etc/wireguard/client_private.key)

sudo tee /etc/wireguard/wg0.conf >/dev/null <<EOF
[Interface]
Address = 10.60.99.10/32
PrivateKey = $CLIENT_PRIVATE_KEY

[Peer]
PublicKey = $GW_PUBLIC_KEY
Endpoint = 172.30.60.1:51820
AllowedIPs = 10.60.99.0/24, 10.60.10.0/24
PersistentKeepalive = 25
EOF

# Xotiradagi o'zgaruvchilarni tozalash:
unset CLIENT_PRIVATE_KEY GW_PUBLIC_KEY

sudo chmod 600 /etc/wireguard/wg0.conf
sudo wg-quick strip wg0 >/dev/null

# Tunnelni ishga tushirish:
sudo systemctl enable --now wg-quick@wg0

# Holatni va marshrutlarni tekshirish:
sudo wg show wg0
ip -br addr show wg0
ip route get 10.60.10.20
ip route get 172.30.60.1
```

> [!TIP]
> **Marshrutlar to‘g‘riligini tekshirish:**
> - `ip route get 10.60.10.20` -> natijada `dev wg0` chiqishi kerak (Split-tunnel orqali ichki tarmoqqa ketadi).
> - `ip route get 172.30.60.1` -> natijada `dev enp0s8` chiqishi kerak (Endpointga bevosita jismoniy tarmoq orqali boradi, hech qanday routing loop yo‘q!).

---

### 8-BOSQICH: VPN-GW da Eng Kam Ruxsatli nftables Xavfsizlik Siyosatini Joriy Etish

**VPN-GW konsolida bajaring:**
Firewall qoidalarini noldan toza yaratamiz:

```bash
# Eski a6_filter jadvali bo'lsa o'chirish:
sudo nft delete table inet a6_filter 2>/dev/null || true

# Yangi filtr jadvalini yaratish:
sudo nft add table inet a6_filter

# INPUT va FORWARD zanjirlarini DEFAULT DROP siyosati bilan yaratish:
sudo nft 'add chain inet a6_filter input { type filter hook input priority 0; policy drop; }'
sudo nft 'add chain inet a6_filter forward { type filter hook forward priority 0; policy drop; }'

# --- INPUT QOIDALARI ---
# 1. Localhostga to'liq ruxsat:
sudo nft add rule inet a6_filter input iifname lo accept

# 2. Allaqachon o'rnatilgan ulanishlarning javob paketlariga ruxsat:
sudo nft add rule inet a6_filter input ct state established,related accept

# 3. Tashqi WAN dan faqat WireGuard shifrlangan portiga (UDP 51820) ruxsat:
sudo nft add rule inet a6_filter input iifname enp0s8 ip saddr 172.30.60.0/24 udp dport 51820 counter accept

# 4. Tashqi WAN dan diagnostika uchun ICMP Echo-request (Ping) ruxsati:
sudo nft add rule inet a6_filter input iifname enp0s8 ip saddr 172.30.60.0/24 icmp type echo-request accept

# 5. Tunnel ichidan Gatewayga ping ruxsati:
sudo nft add rule inet a6_filter input iifname wg0 ip saddr 10.60.99.10 icmp type echo-request accept

# --- FORWARD (TRANZIT) QOIDALARI ---
# 6. Ruxsat berilgan tranzit sessiyalarning qaytuvchi javob paketlariga ruxsat:
sudo nft add rule inet a6_filter forward ct state established,related counter accept

# 7. BEVOSITA WAN->LAN HARAKATLARINI BLOKLASH VA SANASH (T7 testi uchun muhim!):
sudo nft add rule inet a6_filter forward iifname enp0s8 oifname enp0s9 ip daddr 10.60.10.20 counter drop

# 8. Tunnel orqali APP-SRV ning faqat 22 (SSH) va 80 (HTTP) portlariga ruxsat:
sudo nft add rule inet a6_filter forward iifname wg0 oifname enp0s9 ip saddr 10.60.99.10 ip daddr 10.60.10.20 tcp dport '{ 22, 80 }' counter accept

# 9. Tunnel orqali APP-SRV ga ICMP Ping ruxsati:
sudo nft add rule inet a6_filter forward iifname wg0 oifname enp0s9 ip saddr 10.60.99.10 ip daddr 10.60.10.20 icmp type echo-request counter accept

# Qoidalar ro'yxatini tekshirish:
sudo nft -a list table inet a6_filter

# Qoidalarni zaxiraga saqlash:
sudo nft list table inet a6_filter | sudo tee /etc/nftables.a6.conf >/dev/null
```

---

### 9-BOSQICH: Funksional va Salbiy Testlarni O‘tkazish (T1–T7)

Barcha sinovlar **REMOTE-CLIENT** konsolida amalga oshiriladi:

```bash
date -Is

# ==============================================================================
# T1 TESTI: VPNsiz holatda ichki xizmat ochilmasligini qayta tekshirish (Baseline)
# ==============================================================================
sudo wg-quick down wg0
curl --noproxy "*" --connect-timeout 3 http://10.60.10.20/ || true
sudo wg-quick up wg0

# ==============================================================================
# T2 TESTI: WireGuard Handshake va Peer aloqasini tekshirish
# ==============================================================================
ping -c 2 10.60.99.1
sudo wg show wg0
# (Kutilgan: latest handshake: bir necha soniya oldin, transfer: bir necha KiB)

# ==============================================================================
# T3 TESTI: Tunnel orqali ichki ICMP va HTTP xizmatiga kirish
# ==============================================================================
ping -c 2 10.60.10.20
curl --noproxy "*" -sS http://10.60.10.20/
# (Kutilgan: ping muvaffaqiyatli, curl natijasida "A6 INTERNAL SERVICE" chiqadi)

# ==============================================================================
# T4 TESTI: SSH Ed25519 kaliti orqali masofadan xavfsiz boshqaruv
# ==============================================================================
ssh -i ~/.ssh/a6_remoteops -o IdentitiesOnly=yes remoteops@10.60.10.20 'hostname; id'
# (Kutilgan: APP-SRV va uid=1001(remoteops) chiqadi, parol so'ramaydi!)

# ==============================================================================
# T5 TESTI: Parol orqali kirish urinishi rad etilishini tekshirish (Hardening)
# ==============================================================================
ssh -vv -o PreferredAuthentications=password -o PubkeyAuthentication=no \
    -o NumberOfPasswordPrompts=0 remoteops@10.60.10.20 true 2>&1 \
    | grep -E 'Authentications that can continue|Permission denied'
# (Kutilgan: Permission denied (publickey). Server parol so'ramaydi ham!)

# ==============================================================================
# T6 TESTI: Split-Tunnel marshrutlari to'g'riligini tekshirish
# ==============================================================================
ip route get 10.60.10.20
ip route get 172.30.60.1
# (Kutilgan: LAN ga dev wg0, Gateway endpointiga esa dev enp0s8)

# ==============================================================================
# T7 TESTI: Tunnelni chetlab o'tib, bevosita WAN->LAN ga ulanish urinishi (Bypass)
# ==============================================================================
# Tunnelni o'chiramiz va qo'lda noqonuniy marshrut qo'shamiz:
sudo wg-quick down wg0
sudo ip route replace 10.60.10.0/24 via 172.30.60.1 dev enp0s8

# Ulanishga urinish (nftables tomonidan bloklanishi kerak):
curl --noproxy "*" --connect-timeout 3 http://10.60.10.20/ || true
ssh -o ConnectTimeout=3 remoteops@10.60.10.20 true || true

# Noqonuniy marshrutni o'chiramiz va tunnelni qayta yoqamiz:
sudo ip route del 10.60.10.0/24
sudo wg-quick up wg0
```

**VPN-GW konsolida T7 natijasida bloklangan paketlar hisoblagichini tekshirish:**
```bash
sudo nft -a list chain inet a6_filter forward
```
*(7-qoidadagi `iifname enp0s8 oifname enp0s9 ... counter drop` qatorida paketlar va baytlar soni oshganini ko‘rasiz!).*

---

### 10-BOSQICH: Paket Darajasidagi Tahlil: tcpdump Yordamida Outer va Inner PCAP Olish

Ushbu bosqichda WireGuard shifrlashini amalda isbotlash uchun paketlar ushlanadi.

#### 1. VPN-GW da ikkita alohida terminal oynasi oching:
- **VPN-GW 1-Terminalda (Tashqi shifrlangan oqimni ushlash):**
  ```bash
  sudo timeout 15 tcpdump -Z root -ni enp0s8 -s0 -w ~/A6_outer.pcap udp port 51820
  ```
- **VPN-GW 2-Terminalda (Ichki ochilgan oqimni ushlash):**
  ```bash
  sudo timeout 15 tcpdump -Z root -ni enp0s9 -s0 -w ~/A6_inner.pcap 'host 10.60.10.20 and (tcp port 22 or tcp port 80)'
  ```

#### 2. Ushbu 15 soniya davomida REMOTE-CLIENT konsolida trafik hosil qiling:
```bash
curl --noproxy "*" -sS http://10.60.10.20/
ssh -i ~/.ssh/a6_remoteops -o IdentitiesOnly=yes remoteops@10.60.10.20 'date -Is'
```

#### 3. VPN-GW da tahlil natijalarini ko‘rish:
```bash
# Outer PCAP tarkibini ko'rish (faqat UDP 51820 chiqishi kerak):
sudo tcpdump -nn -r ~/A6_outer.pcap -c 8

# Inner PCAP tarkibini ko'rish (TCP 80 va TCP 22 ko'rinadi):
sudo tcpdump -nn -r ~/A6_inner.pcap -c 8

# Xesh summalarni hisoblash:
sha256sum ~/A6_outer.pcap ~/A6_inner.pcap
```

---

### 11-BOSQICH: Konfiguratsiya Auditi, Hisobot Paketini Yig‘ish, SHA-256 va Rollback

> [!NOTE]
> Barcha fayllarni Ubuntu hostdagi umumiy katalogga yoki VM ichidagi `~/A6_hisobot` katalogiga yig‘amiz.

#### 1. VPN-GW da audit fayllarini tayyorlash:
```bash
mkdir -p ~/A6_hisobot

# Maxfiy kalitni yashirgan holda holatni saqlash:
sudo wg show wg0 | sed 's/private key: .*/private key: (hidden)/' | tee ~/A6_hisobot/A6_wg_status.txt

# nftables qoidalarini saqlash:
sudo nft -a list table inet a6_filter | tee ~/A6_hisobot/A6_nft_ruleset.txt

# PCAP fayllarni nusxalash:
cp ~/A6_outer.pcap ~/A6_inner.pcap ~/A6_hisobot/
```

#### 2. APP-SRV da SSH samarali sozlamalarini saqlash:
```bash
mkdir -p ~/A6_hisobot
sudo sshd -T 2>/dev/null \
  | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers|maxauthtries' \
  | tee ~/A6_hisobot/A6_sshd_effective.txt
```

#### 3. Barcha fayllarni bir joyga to‘plab, SHA-256 ro‘yxatini yaratish (Ubuntu Linux Terminalida):
```bash
cd ~/A6_hisobot
sha256sum A6_* > A6_SHA256SUMS.txt
cat A6_SHA256SUMS.txt
```

#### 4. Xavfsiz Tizimni Tiklash (Rollback):
Laboratoriya tugagach, barcha qoidalar va ulanishlar to‘xtatiladi:
- **VPN-GW da:**
  ```bash
  sudo nft delete table inet a6_filter
  sudo systemctl disable --now wg-quick@wg0
  sudo rm -f /etc/sysctl.d/60-a6-vpn.conf
  sudo sysctl -w net.ipv4.ip_forward=0
  ```
- **REMOTE-CLIENT da:**
  ```bash
  sudo systemctl disable --now wg-quick@wg0
  ```

---

## 6. HAR BIR CLI BUYRUQNING MUKAMMAL TAHLILI

Laboratoriya davomida ishlatilgan har bir buyruq va uning parametrlari batafsil tahlili:

### 1. WireGuard Boshqaruvi Buyruqlari:
- `wg genkey`:
  - Curve25519 algoritmi bo‘yicha 256-bitli (32-bayt base64 kodlangan) yangi tasodifiy maxfiy kalit (`private key`) generatsiya qiladi va standart chiqishga (`stdout`) chiqaradi.
- `wg pubkey`:
  - Standart kirishdan (`stdin`) maxfiy kalitni qabul qilib, unga mos keluvchi ochiq kalitni (`public key`) hisoblab chiqaradi.
- `wg show wg0`:
  - `wg0` interfeysining joriy operatsion holatini, portini, ochiq kalitini, ulangan peerlar ro‘yxatini, ularning `latest handshake` (oxirgi ulanish vaqti) hamda `transfer` (RX/TX uzatilgan va qabul qilingan baytlar soni) ko‘rsatkichlarini chiqaradi.
- `wg-quick strip wg0`:
  - `/etc/wireguard/wg0.conf` konfiguratsiya faylini sintaktik tekshiradi va undagi `wg-quick` ga xos direktivalarni (`Address`, `DNS` va h.k.) olib tashlab, sof `wg` utilitasi tushunadigan formatda ekranga chiqaradi (xatoliklarni oldindan topish uchun qulay).
- `wg-quick up wg0` / `wg-quick down wg0`:
  - `up`: Virtual `wg0` interfeysini yaratadi, IP manzil biriktiradi, marshrutlash jadvaliga `AllowedIPs` yo‘nalishlarini qo‘shadi va interfeysni ko‘taradi.
  - `down`: Barcha qo‘shilgan marshrutlarni o‘chiradi va `wg0` interfeysini yadro xotirasidan xavfsiz tozalaydi.
- `systemctl enable --now wg-quick@wg0`:
  - `enable`: Tizim yoqilganda ushbu tunnel avtomatik ko‘tarilishini sozlaydi (systemd unit).
  - `--now`: Buyruq berilgan zahotiyoq xizmatni darhol ishga tushiradi.

### 2. Tarmoq va IP Boshqaruvi (`iproute2`):
- `ip -br link`:
  - `-br` (`--brief`): Interfeyslar ro‘yxatini ixcham, bir qatorli jadval ko‘rinishida chiqaradi (Interfeys nomi, UP/DOWN holati, MAC manzil).
- `ip addr flush dev enp0s8`:
  - `flush`: Ko‘rsatilgan qurilmadagi barcha eski IP manzillarni (masalan, DHCP dan olingan) to‘liq tozalaydi.
- `ip addr add 172.30.60.1/24 dev enp0s8`:
  - `add`: Interfeysga yangi statik IP manzil va tarmoq maskasini (`/24`) biriktiradi.
- `ip link set enp0s8 up`:
  - Tarmoq interfeysining dasturiy linkini faollashtiradi (yoqadi).
- `ip route replace default via 10.60.10.1`:
  - `replace`: Standart shlyuz (gateway) marshrutini xatosiz yangisiga almashtiradi.
- `ip route get 10.60.10.20`:
  - Berilgan manzilga paket qaysi interfeys (`dev`) va qaysi manba IP (`src`) orqali chiqib ketishini simulyatsiya qilib ko‘rsatadi.

### 3. Yadro Parametrlari (`sysctl`):
- `sysctl -w net.ipv4.ip_forward=1`:
  - `-w`: Linux yadrosida IPv4 paketlarni bir tarmoq kartasidan ikkinchisiga uzatish (marshrutizatsiya) imkoniyatini darhol yoqadi.
- `tee /etc/sysctl.d/60-a6-vpn.conf`:
  - Tizim qayta yuklanganda ham `ip_forward=1` sozlamasi saqlanib qolishi uchun konfiguratsiyani doimiy faylga yozadi.

### 4. nftables Xavfsizlik Devori Buyruqlari:
- `nft add table inet a6_filter`:
  - `inet`: IPv4 va IPv6 paketlarni bir vaqtda qayta ishlovchi oila (family).
  - `add table`: `a6_filter` nomli yangi filtrlar jadvalini yaratadi.
- `nft 'add chain inet a6_filter input { type filter hook input priority 0; policy drop; }'`:
  - `hook input`: Bevosita shlyuzning o‘ziga kelayotgan paketlarni tutib oladi.
  - `policy drop`: Qoidalarga to‘g‘ri kelmagan barcha paketlarni sukut bo‘yicha tashlab yuboradi (Default Deny).
- `nft 'add chain inet a6_filter forward { type filter hook forward priority 0; policy drop; }'`:
  - `hook forward`: Shlyuz orqali tranzit o‘tuvchi paketlarni nazorat qiladi.
- `ct state established,related counter accept`:
  - `ct state`: Connection Tracking (ulanishlarni kuzatish) tizimi.
  - `established,related`: Oldinroq ruxsat berilgan ulanishlarga javoban qaytayotgan paketlarni avtomatik o‘tkazib yuboradi.
- `iifname enp0s8 oifname enp0s9 ip daddr 10.60.10.20 counter drop`:
  - `iifname`: Kiruvchi interfeys (`enp0s8` WAN).
  - `oifname`: Chiquvchi interfeys (`enp0s9` LAN).
  - `counter`: Qoidaga tushgan paketlar va baytlar sonini sanab boradi.
  - `drop`: Tunnelni aylanib o‘tishga uringan paketni darhol o‘chiradi.
- `nft -a list table inet a6_filter`:
  - `-a` (`--handle`): Qoidalarning unikal identifikator raqamlarini (`handle`) ko‘rsatgan holda barcha jadvalni chiqaradi.

### 5. Paketlarni Tutish (`tcpdump`):
- `tcpdump -Z root -ni enp0s8 -s0 -w ~/A6_outer.pcap udp port 51820`:
  - `-Z root`: Jarayon huquqlarini root sifatida ushlab turadi (faylga yozishda permission xatosi chiqmasligi uchun).
  - `-n`: IP manzillarni DNS nomlarga aylantirmaydi (tezlikni oshiradi).
  - `-i enp0s8`: Faqat ko‘rsatilgan jismoniy tarmoq interfeysidagi trafikni tinglaydi.
  - `-s0` (`--snapshot-length=0`): Paketlarni kesib tashlamasdan, butun hajmi (to‘liq payload) bilan yozib oladi.
  - `-w <fayl>`: Natijani ekranga emas, Wireshark o‘qiydigan PCAP binar fayliga yozadi.
  - `udp port 51820`: Faqat WireGuard trafigini filtrlaydi.
- `timeout 15`:
  - 15 soniyadan so‘ng tcpdump jarayonini avtomatik to‘xtatadi.

### 6. OpenSSH va Kriptografiya Buyruqlari:
- `ssh-keygen -t ed25519 -a 64 -f ~/.ssh/a6_remoteops -N ''`:
  - `-t ed25519`: Eng xavfsiz va tezkor Ed25519 elliptik egri chizig‘i algoritmi.
  - `-a 64`: KDF (Key Derivation Function) orqali parolni 64 marta xeshlash (brute-forcega qarshi himoya).
  - `-f <fayl>`: Kalit saqlanadigan fayl yo‘li.
  - `-N ''`: Kalitni parolsiz (passphrase yo‘q) yaratish.
- `sshd -t`:
  - `/etc/ssh/sshd_config` fayli sintaksisini tekshiradi. Agar sintaktik xato bo‘lsa, ko‘rsatadi (xizmatni xato bilan o‘chirib qo‘ymaslik uchun juda muhim).
- `sshd -T`:
  - Barcha konfiguratsiya fayllari va drop-in kataloglarini birlashtirgan holda, SSH serverining amaldagi yakuniy konfiguratsiyasini chiqaradi.

### 7. Sinov va Diagnostika Buyruqlari:
- `curl --noproxy "*" --connect-timeout 3 http://10.60.10.20/`:
  - `--noproxy "*"`: Har qanday tizim proksi o‘zgaruvchilarini chetlab o‘tib, to‘g‘ridan-to‘g‘ri ulanadi.
  - `--connect-timeout 3`: Agar 3 soniya ichida server javob bermasa, qotib turmasdan jarayonni to‘xtatadi.
- `ss -lunp | grep ':51820'`:
  - `-l`: Tinglanayotgan (listening) soketlar.
  - `-u`: Faqat UDP protokoli.
  - `-n`: Raqamli portlar ko‘rinishida.
  - `-p`: Qaysi dastur (process) portni egallab turganini ko‘rsatadi.
- `sha256sum <fayllar>`:
  - Fayllarning 256-bitli SHA-2 xeshini hisoblaydi (fayllarning o‘zgarmaganligini va butunligini kafolatlaydi).

---

## 7. NATIJALARNI QAYD ETISH JADVALLARI VA HISOBOT NAMUNASI

Hisobotingiz qabul qilinishi uchun quyidagi jadvallarni o‘z amaliy natijalaringiz bilan to‘ldiring:

### 7.1. Muhit va Tayyorgarlik Jadvali

| Obyekt | Kutilgan Qiymat | Amaldagi Qiymat | Dalil ID | Holati |
| :--- | :--- | :--- | :---: | :---: |
| **VPN-GW interfeyslari** | `172.30.60.1` (WAN); `10.60.10.1` (LAN) | `172.30.60.1; 10.60.10.1` | P1 | OK |
| **REMOTE-CLIENT** | `172.30.60.10` (WAN); `10.60.99.10` (wg0) | `172.30.60.10; 10.60.99.10` | P1 | OK |
| **APP-SRV** | `10.60.10.20`; Gateway: `10.60.10.1` | `10.60.10.20; GW 10.60.10.1` | P1 | OK |
| **WireGuard porti** | `wg0`; UDP port 51820 | `51820/udp listening` | P2 | OK |
| **Kalitlar ruxsati** | Private: `600`; Public: `644` | `-rw------- (600)` | P3 | OK |
| **SSHD sintaksisi** | `sshd -t` exit code: `0` | `Exit: 0 (No syntax errors)` | P4 | OK |

### 7.2. WireGuard va Marshrut Holati Jadvali

| Ko‘rsatkich | Kutilgan Holat | Amaldagi Qiymat | Dalil ID | Izoh |
| :--- | :--- | :--- | :---: | :--- |
| **GW public key** | Private ko‘rinmaydi, 32 bayt public | `<GW_PUBLIC_KEY>` | W1 | Ochiq kalit to‘g‘ri o‘rnatilgan |
| **Client public key** | Private ko‘rinmaydi, 32 bayt public | `<CLIENT_PUBLIC_KEY>` | W2 | Peer sifatida ro‘yxatga olingan |
| **Latest handshake** | Joriy vaqtga yaqin (bir necha soniya) | `latest handshake: 4 seconds ago` | W3 | Tunnel muvaffaqiyatli faol |
| **Transfer RX / TX** | 0 dan katta (baytlar uzatilgan) | `transfer: 12.4 KiB received, 8.2 KiB sent` | W4 | Trafik ikki tomonlama o‘tmoqda |
| **10.60.10.20 route** | `dev wg0` (Tunnel orqali) | `10.60.10.20 dev wg0 table 51820` | W5 | Split-tunnel ishladi |
| **172.30.60.1 route** | `dev enp0s8` (Jismoniy WAN) | `172.30.60.1 dev enp0s8 src 172.30.60.10` | W6 | Routing loop yo‘q |

### 7.3. Funksional va Xavfsizlik Testlari Natijalari (T1–T7)

| Test ID | Test Nomi | Kutilgan Natija | Amaldagi Servis Natijasi | Marshrut / nftables Holati | Xulosa |
| :---: | :--- | :--- | :--- | :--- | :---: |
| **T1** | VPNsiz HTTP ulanish | **Fail** (Timeout / No route) | `curl: (28) Failed to connect` | Ichki tarmoqqa yo‘l yo‘q | **PASSED** |
| **T2** | WG Handshake va Ping | **Success** (Ping va Handshake bor) | `2 packets received, 0% packet loss` | `latest handshake` qayd etildi | **PASSED** |
| **T3** | VPN orqali HTTP / ICMP | **Success** (Veb sahifa ochiladi) | `HTTP 200 OK: "A6 INTERNAL SERVICE"` | `dev wg0` orqali ruxsat berildi | **PASSED** |
| **T4** | SSH Ed25519 kalit bilan | **Success** (Parolsiz kiradi) | `APP-SRV; uid=1001(remoteops)` | SSH kalit autentifikatsiyasi ishladi | **PASSED** |
| **T5** | SSH Parol orqali kirish | **Denied** (Parol taklif etilmaydi) | `Permission denied (publickey)` | Parol bilan kirish to‘liq yopilgan | **PASSED** |
| **T6** | Split-Tunnel marshruti | **LAN: wg0, Endpoint: WAN** | `10.60.10.20 dev wg0; 172.30.60.1 dev enp0s8` | Marshrutlar ajratilgan | **PASSED** |
| **T7** | Direct Bypass urinishi | **Blocked** (Paketlar drop qilinadi) | `curl: (28) Connection timed out` | nftables `counter drop` oshdi | **PASSED** |

### 7.4. Outer va Inner Trafik Taqqoslanishi Jadvali

| Fayl Nomi | Tinglangan Interfeys | Manba IP -> Maqsad IP | Protokol va Port | Ma’lumot Tahlili (Payload) |
| :--- | :--- | :--- | :---: | :--- |
| **A6_outer.pcap** | `enp0s8` (WAN) | `172.30.60.10 -> 172.30.60.1` | **UDP 51820** | WireGuard Noise protokoli. Barcha foydali yuk to‘liq shifrlangan, ochiq matn yo‘q. |
| **A6_inner.pcap** | `enp0s9` (LAN) | `10.60.99.10 -> 10.60.10.20` | **TCP 80** | HTTP GET va "A6 INTERNAL SERVICE" HTML javob matni ochiq ko‘rinadi. |
| **A6_inner.pcap** | `enp0s9` (LAN) | `10.60.99.10 -> 10.60.10.20` | **TCP 22** | SSHv2 protokoli seansi. |
| **A6_SHA256SUMS.txt**| Host tizim | Barcha eksport fayllari | SHA-256 Xesh | Fayllarning kriptografik yaxlitligi tasdiqlandi. |

---

### 7.5. Talabaning Yakuniy Xulosasi (Namunaviy Matn)

> Ushbu A6 amaliy laboratoriya mashg‘ulotida izolyatsiyalangan Linux virtual muhitida WireGuard protokoli negizida masofaviy xavfsiz ulanish (Remote Access VPN) tizimi muvaffaqiyatli loyihalandi va sinovdan o‘tkazildi. T1 bazaviy testi orqali VPN ulanishisiz ichki tarmoqdagi resurslarga bevosita kirish imkonsiz ekanligi isbotlandi. WireGuard shlyuzi va masofaviy mijoz o‘rtasida assimetrik kalitlar almashinib, 1-RTT Noise protokoli asosida faol handshake va xavfsiz tunnel o‘rnatildi. Split-tunnel arxitekturasi yordamida faqat ichki korporativ tarmoq (`10.60.10.0/24`) VPN orqali yo‘naltirildi, bu esa shlyuzga tushadigan ortiqcha yuklama va marshrutlash ilmoqlari (routing loops) xavfini bartaraf etdi. VPN-GW da joriy etilgan nftables xavfsizlik siyosati eng kam ruxsatlar (least privilege) tamoyiliga muvofiq, mijozga faqat ruxsat berilgan portlarga (TCP 22, 80 va ICMP) kirish imkonini berdi hamda tunnelni aylanib o‘tishga qaratilgan bevosita WAN->LAN harakatlarini qat’iy blokladi. Ichki serverda SSH xizmati kuchaytirilib (hardening), parolli autentifikatsiya to‘liq taqiqlandi va faqat Ed25519 ochiq kaliti orqali kirish ta’minlandi. Olingan PCAP fayllari tahlili tashqi segmentda trafik faqat shifrlangan UDP/51820 holatida harakatlanishini to‘liq tasdiqladi.

---

## 8. O‘QITUVCHI UCHUN HIMOYA SAVOLLARI VA MUKAMMAL JAVOBLAR

### 1-savol: WireGuard endpoint IP bilan tunnel IP o‘rtasidagi farq nima?
**Mukammal javob:**
- **Endpoint IP (`172.30.60.1:51820`):** Bu jismoniy yoki tashqi marshrutlanadigan tarmoq manzilidir. Shifrlangan UDP paketlar internet yoki oraliq tarmoq orqali aynan shu IP va portga yetkaziladi. U haqiqiy tarmoq kartasiga (`enp0s8`) biriktirilgan bo‘ladi.
- **Tunnel IP (`10.60.99.1` va `10.60.99.10`):** Bu faqat shifrlangan overlay (virtual) tarmoq ichida mavjud bo‘lgan mantiqiy IP manzil bo‘lib, `wg0` virtual interfeysiga biriktiriladi. Ushbu manzil tashqi tarmoqda ko‘rinmaydi va paket shifrdan yechilgandan so‘nggina ichki tizimlar tomonidan qayta ishlanadi.

---

### 2-savol: Nega VPN-GW peeridagi AllowedIPs 10.60.99.10/32 bilan cheklangan?
**Mukammal javob:**
Bu WireGuardning **Cryptokey Routing** xavfsizlik tamoyilidir. Shlyuz faqatgina ushbu mijozga tegishli aniq bitta manzilni (`10.60.99.10/32`) ro‘yxatga oladi. Agar masofaviy mijoz o‘zining IP manzilini soxtalashtirib (IP Spoofing qilib), boshqa biror IP manzil nomidan paket yuborishga urinsa, WireGuard yadrosi kelgan paketning `Source IP` manzili ushbu peerning `AllowedIPs` qiymatiga to‘g‘ri kelmasligini ko‘radi va paketni zudlik bilan tashlab yuboradi (`drop`).

---

### 3-savol: REMOTE-CLIENT dagi AllowedIPs marshrut yaratish va peer tanlashga qanday ta’sir qiladi?
**Mukammal javob:**
Mijoz tomonda `AllowedIPs = 10.60.99.0/24, 10.60.10.0/24` ko‘rsatilganda ikki muhim hodisa ro‘y beradi:
1. `wg-quick` vositasi Linux yadrosi marshrutlash jadvaliga (`ip route`) qoidalar qo‘shadi: `10.60.99.0/24` va `10.60.10.0/24` tarmoqlariga boruvchi paketlar `wg0` interfeysiga yo‘naltirilsin.
2. WireGuard kriptografik darajada: ushbu tarmoqlarga yuborilayotgan barcha paketlarni aynan VPN-GW ning ochiq kaliti bilan shifrlash kerakligini biladi.

---

### 4-savol: 0.0.0.0/0 o‘rniga split-tunnel prefixlari tanlanishining afzalligi va riski nima?
**Mukammal javob:**
- **Afzalligi:** VPN shlyuzi va korporativ kanal ortiqcha yuklanmaydi. Foydalanuvchining oddiy internet trafigi (masalan, YouTube, fayl yuklash) to‘g‘ridan-to‘g‘ri o‘z provayderi orqali ketadi, bu esa tezlikni oshiradi va server resurslarini tejaydi.
- **Riski:** Foydalanuvchi kompyuteri bir vaqtning o‘zida ham tashqi ochiq internetga, ham himoyalangan ichki tarmoqqa ulangan bo‘ladi. Agar foydalanuvchi kompyuteri tashqi internetdan zararlansa (malware tushsa), bu kompyuter ichki tarmoqqa hujum qilish uchun tramplin (pivot point) vazifasini o‘tab berishi mumkin.

---

### 5-savol: Endpoint marshruti wg0 ga tushib qolsa qanday routing loop yuzaga kelishi mumkin?
**Mukammal javob:**
Agar xatolik tufayli endpoint manzili (`172.30.60.1`) ham tunnel orqali yo‘naltirilsa:
1. WireGuard shifrlangan UDP paketni `172.30.60.1:51820` ga jo‘natish uchun yadroga uzatadi.
2. Yadro marshrutlar jadvaliga qaraydi va `172.30.60.1` ga borish uchun paketni yana `wg0` ga qaytaradi.
3. WireGuard ushbu UDP paketni yana yangi paket deb o‘ylab, uning ustiga yana bitta shifr qatlamini o‘raydi va yana yadroga beradi.
Bu cheksiz sikl (**Routing Loop**) hosil qilib, tizim xotirasi va kanalni bir lahzada to‘ldiradi hamda aloqa butunlay uziladi.

---

### 6-savol: Latest handshake borligi APP-SRV xizmati albatta ishlashini isbotlaydimi?
**Mukammal javob:**
**Yo‘q, mutlaqo isbotlamaydi!**
`Latest handshake` faqatgina `REMOTE-CLIENT` bilan `VPN-GW` o‘rtasida WireGuard 1-qatlam ulanishi muvaffaqiyatli bo‘lganini va kalitlar tasdiqlanganini bildiradi. Ammo bu APP-SRV xizmatlari ishlashini kafolatlamaydi, chunki:
- VPN-GW da IP forwarding o‘chirilgan bo‘lishi mumkin;
- nftables tranzit paketlarni bloklayotgan bo‘lishi mumkin;
- APP-SRV mashinasi o‘chgan yoki undagi Nginx/SSH servislari to‘xtab qolgan bo‘lishi mumkin.

---

### 7-savol: Nega WireGuard tunnelining o‘zi SSH parol autentifikatsiyasini xavfsiz deb qo‘ymaydi?
**Mukammal javob:**
Chunki WireGuard faqat tarmoq darajasidagi transport shifrlashini ta’minlaydi. Agar masofaviy kompyuterga tajovuzkor kirib olsa yoki mijoz qurilmasi xakerlar tomonidan boshqarilsa, u tunnel orqali ichki serverdagi SSH ga soniyasiga yuzlab parollar jo‘natib, **Brute-Force** yoki **Dictionary Attack** uyushtirishi mumkin. Shu sababli, **Defense in Depth** talabiga binoan, SSH darajasida parol bilan kirish butunlay o‘chirilib, faqat Ed25519 shaxsiy kaliti bilan kirish qoldirilishi shart.

---

### 8-savol: Private kalit 600, authorized_keys esa 600 bo‘lishi nima uchun muhim?
**Mukammal javob:**
- Linux fayl tizimida `chmod 600` (`-rw-------`) huquqi faylni faqat uning egasi o‘qiy va yozishi mumkinligini bildiradi.
- Agar private kalitga `644` yoki `777` ruxsat berilsa, ushbu operatsion tizimdagi har qanday oddiy yoki zararli dastur (yoki boshqa foydalanuvchi) maxfiy kalitni o‘g‘irlab oladi.
- OpenSSH dasturi xavfsizlik yuzasidan o‘rnatilgan qoidaga ko‘ra, agar `authorized_keys` yoki `id_ed25519` fayli ruxsati juda ochiq bo‘lsa (`chmod 644/777`), ulanishni xavfli deb hisoblaydi va avtomatik ravishda **rad etadi (Ignore / Permission Denied)**.

---

### 9-savol: PersistentKeepalive=25 qachon kerak va qanday trade-off keltiradi?
**Mukammal javob:**
- **Qachon kerak:** Masofaviy mijoz uy Wi-Fi routeri yoki NAT / Stateful Firewall ortida joylashgan bo‘lsa kerak bo‘ladi. Routerlar odatda 30–60 soniya harakatsizlikdan so‘ng o‘zlarining NAT sessiya jadvalidagi port yo‘naltirishlarini (state table) tozalab tashlaydi. `PersistentKeepalive = 25` har 25 soniyada shlyuzga 32 baytli kichik bo‘sh paket yuborib, NAT portini doimiy ochiq saqlaydi.
- **Trade-off (Kamchiligi):** Har 25 soniyada tarmoq trafigi sarflanadi va mobil qurilmalarda batareya quvvatini tezroq sarflaydi.

---

### 10-savol: WAN->LAN direct drop counteri qaysi hujum yoki xato yo‘lni ko‘rsatadi?
**Mukammal javob:**
Bu hisoblagich tajovuzkor yoki xato sozlangan mijoz WireGuard VPN tunnelini chetlab o‘tib, to‘g‘ridan-to‘g‘ri A6-WAN interfeysi orqali ichki A6-LAN tarmog‘idagi `10.60.10.20` manziliga marshrut o‘rnatib kirishga uringanini ko‘rsatadi. Bu qoida shlyuz orqali ruxsatsiz tranzit o‘tish (Unauthorized Routing / Bypass) urinishlarini fosh qiladi va bloklaydi.

---

### 11-savol: Outer va inner PCAP manba/maqsadlari nima uchun farq qiladi?
**Mukammal javob:**
Chunki WireGuard kapsulyatsiya qiladi:
- **Outer PCAP:** Tarmoqlararo jismoniy paketlarni ko‘rsatadi. Unda IP manba `172.30.60.10` va IP maqsad `172.30.60.1` bo‘lib, protokoli **UDP port 51820** bo‘ladi.
- **Inner PCAP:** Shlyuz shifrni yechib, paketni ichki interfeysga uzatgandan keyingi holatni ko‘rsatadi. Unda mantiqiy IP manba `10.60.99.10` va IP maqsad `10.60.10.20` bo‘lib, protokoli **TCP 22** yoki **TCP 80** ko‘rinishida bo‘ladi.

---

### 12-savol: VPN muhitiga MFA va qurilma posture nazorati qayerda qo‘shilishi mumkin?
**Mukammal javob:**
WireGuardning o‘zi sof yadro sathidagi protokoli bo‘lib, unda interaktiv login/parol yoki MFA (ko‘p omilli autentifikatsiya) oynasi yo‘q. Shu sababli:
1. **SSO / OIDC integratsiyasi:** WireGuard ustiga qurilgan zamonaviy boshqaruv platformalari (masalan, **Tailscale**, **NetBird**, **Firezone**) orqali foydalanuvchiga Google/Okta/Keycloak bilan 2FA/MFA orqali autentifikatsiya kiritiladi.
2. **Qurilma posture nazorati (Device Posture Check):** Mijoz qurilmasida maxsus agent ishlab, tizimda antivirus borligi, disk shifrlangani (BitLocker/LUKS) va xavfsizlik yangilanishlari o‘rnatilganini tekshiradi va faqat shundan keyingina WireGuard kalitini faollashtiradi.

---

### 13-savol: Kalit komprometatsiyasida peer kirishini bekor qilish va rotatsiya tartibi qanday?
**Mukammal javob:**
Agar masofaviy xodimning xususiy kaliti yoki noutbuki o‘g‘irlansa:
1. **Darhol bekor qilish (Revocation):** VPN-GW shlyuzidagi `/etc/wireguard/wg0.conf` faylidan ushbu mijozning `[Peer]` bloki (uning `PublicKey`) o‘chirib tashlanadi va `wg syncconf wg0 <(wg-quick strip wg0)` buyrug‘i beriladi. Shlyuz ushbu kalitdan kelgan paketlarni darhol inkor qiladi.
2. **SSH kalitini o‘chirish:** Ichki serverlardagi `~/.ssh/authorized_keys` faylidan ushbu mijozning ochiq kaliti olib tashlanadi.
3. **Rotatsiya:** Xodimga yangi kalit juftligi generatsiya qilinadi va yangi `PublicKey` tizimga qo‘shiladi.

---

### 14-savol: Ushbu laboratoriyani ishlab chiqarish muhitiga ko‘chirishdan oldin nimalar baholanadi?
**Mukammal javob:**
1. **Markazlashtirilgan Kalitlar Boshqaruvi:** Kalitlarni qo‘lda almashish o‘rniga avtomatlashtirilgan PKI yoki boshqaruv tekisligi (Control Plane) joriy etish.
2. **Monitoring va SIEM:** VPN sessiyalari va ulanish urinishlarini markaziy SIEM tizimiga (Splunk, Wazuh, ELK) yuborish.
3. **Zaxiralash va Yuqori Mavjudlik (High Availability):** Shlyuzning ikkita nusxasini ko‘tarib, Keepalived (VRRP) yoki BGP orqali nosozlikka bardoshli (failover) qilish.
4. **MFA va SSO:** Xodimlar kirishi uchun 2FA/MFA tizimini majburiy qilish.

---

## 9. FOYDALANILGAN RASMIY MANBALAR

1. **WireGuard Rasmiy Hujjatlari:**
   - WireGuard Installation & Quickstart: https://www.wireguard.com/quickstart/
   - WireGuard Protocol & Cryptography: https://www.wireguard.com/protocol/
   - WireGuard Formal Verification: https://www.wireguard.com/formal-verification/
2. **Kriptografik Standartlar va RFC hujjatlari:**
   - RFC 7539: ChaCha20 and Poly1305 for IETF Protocols: https://datatracker.ietf.org/doc/html/rfc7539
   - RFC 7748: Elliptic Curves for Security (Curve25519): https://datatracker.ietf.org/doc/html/rfc7748
   - Noise Protocol Framework: https://noiseprotocol.org/
3. **Linux Yadro va Tarmoq Ma’lumotnomalari:**
   - wireguard-tools man page: `man 8 wg`, `man 8 wg-quick` (https://man7.org/linux/man-pages/man8/wg-quick.8.html)
   - nftables Wiki — Server xavfsizlik devori: https://wiki.nftables.org/wiki-nftables/index.php/Simple_ruleset_for_a_server
   - OpenSSH Server konfiguratsiya qo‘llanmasi: https://man.openbsd.org/sshd_config
   - Linux Advanced Routing & Traffic Control (LARTC): https://lartc.org/
