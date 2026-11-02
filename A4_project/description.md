# TARMOQ XAVFSIZLIGI — A4 AMALIY MASHG‘ULOTI QO‘LLANMASI
**1-VARIANT: Stateful nftables siyosati, LAN uchun SNAT va DMZ veb-serveri uchun DNAT**

---

> **DIQQAT — ISH MUHITI VA MAXSUS SHARTLAR:**
> 1. **Lokal Wi-Fi tarmog‘i:** Ushbu laboratoriya universitet tarmog‘ida emas, **shaxsiy / uy lokal Wi-Fi** tarmog‘ida bajariladi. Shu sababli universitet proksisi (`172.18.1.1:2500`) yoki fayl serveri (`\\172.18.233.234`) **ishlatilmaydi**. Virtual mashinalar paketlarni yuklashda to‘g‘ridan-to‘g‘ri internetga chiqadi.
> 2. **Operatsion tizim (Host):** Asosiy kompyuter — **Ubuntu Linux** (Windows emas!). Virtualizatsiya vositasi — **VMware Workstation Pro / Player for Linux**.
> 3. **Virtual mashinalar (Guest):** VMware ichida **4 ta Ubuntu Server 24.04 LTS** virtual mashinasi ishga tushiriladi (`FW-RTR`, `LAN-CLIENT`, `WEB-DMZ`, `EXT-1`).
> 4. **CLI komandalar tushuntirishi:** Qo‘llanmadagi har bir terminal buyrug‘i, uning parametrlari (flaglari) va nima vazifani bajarishi qadamma-qadam to‘liq tushuntirilgan.

---

## MUNDARIJA
1. [1. Mavzuning Asosiy Maqsadi va Vazifalari](#1-mavzuning-asosiy-maqsadi-va-vazifalari)
2. [2. Muhitni Sozlash: Ubuntu Host, VMware va Lokal Wi-Fi](#2-muhitni-sozlash-ubuntu-host-vmware-va-lokal-wi-fi)
3. [3. Har Bir Mavzu Bo‘yicha Mukammal Nazariy Qo‘llanma](#3-har-bir-mavzu-boyicha-mukammal-nazariy-qollanma)
   - 3.1. Netfilter va nftables arxitekturasi
   - 3.2. Stateful va Stateless filtrlash farqi (Connection Tracking)
   - 3.3. NAT texnologiyasi: SNAT va DNAT qanday ishlaydi?
   - 3.4. Default Deny va Minimal Ruxsatlar tamoyili
   - 3.5. Log va Counter mexanizmlari, Rate-limiting
4. [4. Tarmoq Topologiyasi, Adreslash va Siyosat Matritsasi](#4-tarmoq-topologiyasi-adreslash-va-siyosat-matritsasi)
5. [5. Mashqni Bajarishning Bosqichma-bosqich Yo‘riqnomasi](#5-mashqni-bajarishning-bosqichma-bosqich-yoriqnomasi)
   - 0-bosqich: Ubuntu hostda VM nusxalarini tayyorlash
   - 1-bosqich: Lokal Wi-Fi orqali paketlarni o‘rnatish va Snapshot olish
   - 2-bosqich: Virtual interfeyslarni aniqlash va FW-RTR routing sozlamalari
   - 3-bosqich: LAN-CLIENT, WEB-DMZ va EXT-1 mijozlarini sozlash
   - 4-bosqich: WEB-DMZ va EXT-1 da sinov xizmatlarini ishga tushirish
   - 5-bosqich: FW-RTR da nftables qoidalarini (ruleset) yozish
   - 6-bosqich: Sintaksisni tekshirish, rulesetni qo‘llash va audit
   - 7-bosqich: 10 ta ijobiy va salbiy testlarni o‘tkazish (T1–T10)
   - 8-bosqich: Paket darajasida tahlil: tcpdump va conntrack dalillari
   - 9-bosqich: Nazoratli xatolik: established,related qoidasini o‘chirish va tahlil
   - 10-bosqich: Dalillarni saqlash va tizimni tozalash
6. [6. Har Bir CLI Buyruqning Mukammal Tahlili](#6-har-bir-cli-buyruqning-mukammal-tahlili)
7. [7. To‘ldiriladigan Natija Jadvallari va Hisobot Namunasi](#7-toldiriladigan-natija-jadvallari-va-hisobot-namunasi)
8. [8. O‘qituvchi Uchun Himoya Savollari va Mukammal Javoblar](#8-oqituvchi-uchun-himoya-savollari-va-mukammal-javoblar)
9. [9. Foydalanilgan Rasmiy Manbalar](#9-foydalanilgan-rasmiy-manbalar)

---

## 1. MAVZUNING ASOSIY MAQSADI VA VAZIFALARI

Ushbu amaliy laboratoriya mashg‘ulotining bosh maqsadi — talabaga zamonaviy Linux yadrosi darajasidagi paketlar filtrlash mexanizmi bo‘lgan **Netfilter / nftables** vositasida professional xavfsizlik devori (stateful firewall) va tarmoq manzillarini tarjima qilish (**NAT — Network Address Translation**) tizimini noldan loyihalash, sozlash, sinash hamda audit qilish ko‘nikmalarini berishdir.

### Asosiy maqsadlar:
1. **Zonal izolyatsiya va marshrutlash:** Uchta virtual segment (`A4-LAN`, `A4-DMZ`, `A4-WAN`) orasida Ubuntu Server 24.04 negizida ishlovchi xavfsiz marshrutizator (`FW-RTR`) yaratish va IPv4 forwardingni yoqish.
2. **"Default Deny" siyosatini joriy etish:** Firewallning `input` va `forward` zanjirlarida barcha trafikni sukut bo‘yicha taqiqlash (`policy drop`) va faqat talab qilingan oqimlarga minimal ruxsat berish.
3. **Stateful Connection Tracking:** Paketlarni shunchaki alohida emas, ularning sessiya kontekstida (`conntrack` yordamida: `NEW`, `ESTABLISHED`, `RELATED`, `INVALID`) tahlil qilish hamda javob paketlarini `ct state established,related accept` orqali xavfsiz o‘tkazish.
4. **SNAT (Source NAT):** Ichki korporativ tarmoq (`LAN: 10.40.10.0/24`) tashqi manbalarga murojaat qilganda ularning xususiy IP manzillarini firewallning tashqi ochiq IP manzili (`198.18.0.1`) ga almashtirib chiqarish.
5. **DNAT / Port Forwarding (Destination NAT):** Tashqi tarmoq (`WAN: EXT-1`) dan firewallning `198.18.0.1:8080` manziliga kelgan so‘rovlarni xavfsiz tarzda DMZ zonasidagi veb-serverning `10.40.30.20:80` manziliga yo‘naltirish.
6. **Ilmiy va amaliy dalillar bilan isbotlash:** Shunchaki "sayt ochildi" deb cheklanmasdan, qoidalar ishlaganini `nftables` counterlari (hisoblagichlar), tizim jurnallari (`journalctl / dmesg`), paketlar ushlagichi (`tcpdump` / `.pcap`) va ulanishlar jadvali (`conntrack`) orqali har tomonlama isbotlash.

---

## 2. MUHITNI SOZLASH: UBUNTU HOST, VMWARE VA LOKAL WI-FI

Universitet laboratoriya sharoitida talabalarga Windows 11 kompyuterlarida ichki SMB serverdan obraz ko‘chirish va universitet proksisi orqali internetga chiqish aytilgan. **Lekin siz o‘z shaxsiy kompyuteringizda (Ubuntu OS) va shaxsiy Wi-Fi tarmog‘ingizda ishlayapsiz.**

Quyidagi o‘zgarishlarga qat’iy rioya qiling:

```
+---------------------------------------------------------------------------------+
| HOST TIZIM: Ubuntu Linux (Desktop)                                              |
| Wi-Fi: Shaxsiy Wi-Fi (To'g'ridan-to'g'ri internetga chiqish, PROKSI YO'Q!)      |
| Virtualizatsiya: VMware Workstation Pro / Player for Linux                      |
|                                                                                 |
|                      +----------------------------------+                       |
|                      |             FW-RTR               |                       |
|                      |      (Ubuntu Server 24.04)       |                       |
|                      +----+-------------+------------+--+                       |
|                           |             |            |                          |
|         +-----------------+             |            +------------------+       |
|         | (A4-LAN segment)              | (A4-DMZ segment)              |       |
|         | 10.40.10.0/24                 | 10.40.30.0/24                 |       |
|         v                               v                               v       |
|  +---------------+               +---------------+               +---------------+
|  |  LAN-CLIENT   |               |    WEB-DMZ    |               |     EXT-1     |
|  | 10.40.10.10   |               |  10.40.30.20  |               | 198.18.0.10   |
|  +---------------+               +---------------+               +---------------+
|                                                           (A4-WAN segment)      |
|                                                            198.18.0.0/24        |
+---------------------------------------------------------------------------------+
```

### Muhim farqlar va qoidalar:
1. **Proksini aslo yozmang:**
   - PDF dagi `export http_proxy="http://172.18.1.1:2500"` va `/etc/apt/apt.conf.d/99proxy` faylini **yaratmang**! Agar yaratsangiz yoki nusxalasangiz, shaxsiy Wi-Fi da `apt update` xatolik beradi (`Connection timed out` yoki `Could not resolve proxy`).
   - Agar ilgari yozilgan bo‘lsa, uni darhol o‘chiring:
     ```bash
     sudo rm -f /etc/apt/apt.conf.d/99proxy
     unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
     ```
2. **Paketlarni o‘rnatish bosqichi:**
   - Har bir virtual mashinada boshlang‘ich bosqichda bitta tarmoq adapterini **VMware NAT** (`vmnet8`) rejimida qoldiring. Bu adapter Ubuntu host orqali sizning uyingizdagi Wi-Fi ga ulanadi va paketlar erkin yuklab olinadi.
   - Paketlar to‘liq o‘rnatib bo‘lingach, bu NAT adapterni virtual mashina sozlamalaridan o‘chirib qo‘yasiz (Disconnect qilasiz).
3. **VMware Workstation for Linux da LAN Segment yaratish:**
   - VMware Workstation oynasida har bir VM ustiga sichqonchaning o‘ng tugmasini bosing -> `Settings` -> `Network Adapter`.
   - `Network connection` qismida `LAN segment` bandini tanlang.
   - `LAN Segments...` tugmasini bosib, 3 ta segment nomini qo‘shing:
     - `A4-LAN`
     - `A4-DMZ`
     - `A4-WAN`
   - Har bir virtual mashina adapterini jadvalga muvofiq tegishli segmentga ulang.

---

## 3. HAR BIR MAVZU BO‘YICHA MUKAMMAL NAZARIY QO‘LLANMA

### 3.1. Netfilter va nftables arxitekturasi
Linux operatsion tizimi yadrosida (Kernel space) tarmoq trafigini tahlil qilish, boshqarish va o‘zgartirish uchun **Netfilter** quyi tizimi mavjud. Ilgari Netfilter bilan ishlash uchun `iptables`, `ip6tables`, `arptables` va `ebtables` kabi alohida utilitalar ishlatilar edi. 

Zamonaviy Linux yadrolarida (jumladan, Ubuntu 24.04 da) bularning barchasi yagona, tezkor va arxitekturaviy jihatdan mukammal bo‘lgan **nftables** bilan almashtirilgan:
- **Jadvallar (Tables):** Qoidalar to‘plamining eng yuqori konteyneri. `inet` (IPv4 va IPv6 birga), `ip` (faqat IPv4) kabi oilalarga ega.
- **Zanjirlar (Chains):** Paketlar o‘tadigan yo‘laklar. Ular ma’lum bir "hook" (ilmoq) ga ulanadi:
  - `prerouting`: Paket tarmoq kartasidan kirib kelgan zahoti, marshrutlashdan (routing decision) oldin tutiladi (DNAT shu yerda qilinadi).
  - `input`: Paket aynan ushbu mashinaning o‘ziga (masalan, FW-RTR dagi SSH xizmatiga) yo‘naltirilgan bo‘lsa ishlaydi.
  - `forward`: Paket ushbu mashinadan tranzit sifatida o‘tib, boshqa tarmoqqa ketayotgan bo‘lsa ishlaydi (LAN -> DMZ, LAN -> WAN).
  - `output`: Mashinaning o‘zi tomonidan yaratilgan chiquvchi paketlar uchun.
  - `postrouting`: Paket marshrutlashdan o‘tib, tarmoq kartasidan chiqib ketishidan oldin ishlaydi (SNAT shu yerda qilinadi).

```
[Paket kirishi] ---> (prerouting / DNAT) ---> [Marshrutlash qarori]
                                                     |
                   +---------------------------------+---------------------------------+
                   | (Paket mashinaning o'ziga)                                        | (Paket tranzit)
                   v                                                                   v
            (input zanjiri)                                                     (forward zanjiri)
                   |                                                                   |
                   v                                                                   v
            [Lokal jarayon]                                                            |
                   |                                                                   |
                   v                                                                   |
            (output zanjiri)                                                           |
                   |                                                                   |
                   +---------------------------------+---------------------------------+
                                                     |
                                                     v
                                          (postrouting / SNAT) ---> [Paket chiqishi]
```

---

### 3.2. Stateful va Stateless filtrlash farqi (Connection Tracking)
Oddiy (**stateless**) firewall har bir paketni oldingi yoki keyingi paketlardan mustaqil ravishda, faqat uning sarlavhasidagi IP va portga qarab tekshiradi. Bu usulda qaytuvchi javob trafigini o‘tkazish uchun teskari barcha dinamik portlarga ruxsat berishga to‘g‘ri keladi, bu esa ulkan xavfsizlik teshigini keltirib chiqaradi.

Zamonaviy (**stateful**) firewall esa Linuxning **conntrack** (Connection Tracking) moduli orqali ulanishlar tarixini va holatini kuzatib boradi:
- `NEW`: Yangi ulanish boshlanmoqda (masalan, birinchi TCP SYN paketi).
- `ESTABLISHED`: Ikki tomonlama ulanish o‘rnatilgan, javob paketlari va ma’lumot almashinuvi ketmoqda.
- `RELATED`: Mavjud ulanishga bog‘liq bo‘lgan yangi ulanish (masalan, FTP data kanali yoki ICMP error xabarlari).
- `INVALID`: Noto‘g‘ri, buzilgan yoki holatlar jadvalida mavjud bo‘lmagan anomal paketlar.

**Nega `ct state established,related accept` qoidasi eng muhim?**
Foydalanuvchi tashqi veb-serverga so‘rov yuborganda (TCP SYN), firewall uni `NEW` sifatida qabul qiladi va ruxsat beradi. Server javob qaytarganda (TCP SYN-ACK), bu paket avtomatik ravishda `ESTABLISHED` holatiga o‘tadi. Agar firewallda ushbu qoida bo‘lsa, qaytuvchi javob uchun alohida tashqi kirish qoidasi yozish shart emas! U avtomatik xavfsiz o‘tkaziladi.

---

### 3.3. NAT texnologiyasi: SNAT va DNAT qanday ishlaydi?

NAT (Network Address Translation) — IP paketlar sarlavhasidagi manzil va port ma’lumotlarini parvoz vaqtida (on-the-fly) o‘zgartirish texnologiyasidir.

#### 1. SNAT (Source NAT):
- **Vazifasi:** Chiquvchi paketning **Manba (Source) IP** manzilini o‘zgartiradi.
- **Qo‘llanishi:** Ichki xususiy tarmoqlar (RFC 1918 bo‘yicha `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) internetda marshrutlanmaydi. LAN-CLIENT (`10.40.10.10`) tashqi EXT-1 (`198.18.0.10`) ga murojaat qilganda, FW-RTR routeri paket manbasini o‘zining tashqi WAN IP si bo‘lgan `198.18.0.1` ga o‘zgartiradi (`postrouting` zanjirida).
- EXT-1 javobni `198.18.0.1` ga qaytaradi, FW-RTR esa `conntrack` jadvaliga qarab, paketni yana `10.40.10.10` ga qayta yo‘naltiradi.

#### 2. DNAT (Destination NAT / Port Forwarding):
- **Vazifasi:** Kiruvchi paketning **Maqsad (Destination) IP va Portini** o‘zgartiradi.
- **Qo‘llanishi:** Xavfsizlik nuqtai nazaridan DMZ zonasidagi ichki serverlar (`10.40.30.20`) tashqi dunyoga to‘g‘ridan-to‘g‘ri ochilmaydi. Tashqi EXT-1 foydalanuvchisi FW-RTR ning WAN manziliga (`198.18.0.1:8080`) so‘rov yuboradi.
- FW-RTR ning `prerouting` zanjiridagi DNAT qoidasi maqsad manzilni darhol `10.40.30.20:80` ga o‘zgartiradi.
- **O‘ta muhim qoida:** DNAT paket manzilini o‘zgartirgani bilan uni tarmoqqa o‘tkazib yubormaydi! DNAT qilingan paket keyin `forward` zanjiriga kiradi. Agar `forward` zanjirida `ct status dnat` ruxsat qoidasi bo‘lmasa, paket **DROP** qilinadi.

---

### 3.4. Default Deny va Minimal Ruxsatlar tamoyili
Xavfsizlik tizimlarida ikkita yondashuv mavjud:
1. *Default Accept (Blacklist):* Hamma narsaga ruxsat beriladi, faqat ma’lum xavfli portlar bloklanadi. Bu juda xavfli, chunki administrator bilmagan har qanday yangi xizmat ochiq qoladi.
2. *Default Deny / Zero Trust (Whitelist):* Hamma narsa sukut bo‘yicha taqiqlanadi (`policy drop`). Faqat biznes va texnik talab bo‘lgan aniq oqimlarga (IP, interfeys, protokol, port ko‘rsatilgan holda) minimal ruxsat beriladi.

Ushbu mashg‘ulotda `input` va `forward` zanjirlari qat’iy `policy drop` rejimiga qo‘yiladi.

---

### 3.5. Log va Counter mexanizmlari, Rate-limiting
Qoidalar samaradorligini isbotlash uchun `nftables` da har bir qoidaga `counter` qo‘shiladi (u nechta paket va nechta bayt ushbu qoidaga tushganini hisoblab boradi).

Taqiqlangan paketlar xavfsizlik monitoringi uchun jurnallanadi (`log prefix "A4-... "`):
- **Rate-limiting zarurati:** Agar tajovuzkor soniyasiga millionlab noto‘g‘ri paket yuborsa (DoS hujumi), firewall ularning barchasini qattiq diskka (log faylga) yozishga harakat qilib, server xotirasini yoki diskini to‘ldirib qo‘yishi mumkin.
- Shuning uchun `limit rate 5/minute` cheklovi o‘rnatiladi: daqiqasiga ko‘pi bilan 5 ta bloklangan paket logga yoziladi, qolganlari esa jimgina tashlab yuboriladi. Bu loglar `journalctl -k` yoki `/var/log/syslog` da aks etadi.

---

## 4. TARMOQ TOPOLOGIYASI, ADRESLASH VA SIYOSAT MATRITSASI

### 4.1. Adreslash Rejasi (Addressing Plan)

| Segment nomi | Subnet (Tarmoq) | Tugun / Rol | Interfeys namunasi | Biriktirilgan IP / Prefiks | Gateway (Shlyuz) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **A4-LAN** | `10.40.10.0/24` | FW-RTR (LAN NIC) | `ens37` (yoki enp0s8) | `10.40.10.1/24` | — |
| | | LAN-CLIENT | `ens33` (yoki enp0s8) | `10.40.10.10/24` | `10.40.10.1` |
| **A4-DMZ** | `10.40.30.0/24` | FW-RTR (DMZ NIC) | `ens38` (yoki enp0s9) | `10.40.30.1/24` | — |
| | | WEB-DMZ | `ens33` (yoki enp0s8) | `10.40.30.20/24` | `10.40.30.1` |
| **A4-WAN** | `198.18.0.0/24` | FW-RTR (WAN NIC) | `ens39` (yoki enp0s10)| `198.18.0.1/24` | — |
| | | EXT-1 | `ens33` (yoki enp0s8) | `198.18.0.10/24` | — (Yo‘q) |

> **Eslatma:** `198.18.0.0/24` tarmog‘i RFC 2544 standarti bo‘yicha tarmoq qurilmalarini benchmark sinovlaridan o‘tkazish uchun ajratilgan maxsus diapazon bo‘lib, u real internetga chiqmaydigan, izolyatsiyalangan tashqi tarmoq vazifasini o‘taydi.

---

### 4.2. Xavfsizlik Siyosati Matritsasi (P1 – P8)

| Talab ID | Manba → Maqsad | Protokol / Port | Qaror (Verdict) | NAT turi | Amalga oshirish sababi (Asos) |
| :---: | :--- | :--- | :---: | :---: | :--- |
| **P1** | LAN → FW-RTR | SSH 22, ICMP Echo | **Accept** | — | FW-RTR ni ichki xavfsiz boshqarish va tarmoq diagnostikasi |
| **P2** | WAN / DMZ → FW-RTR | Barcha yangi oqimlar | **Drop** | — | Firewall boshqaruv panelini tashqi va oraliq xavflardan yopish |
| **P3** | LAN → WAN (EXT-1) | TCP 8000 | **Accept** | **SNAT** | Ichki mijozning tashqi ruxsat berilgan xizmatga xavfsiz chiqishi |
| **P4** | LAN → WAN (EXT-1) | TCP 9000 | **Drop** | — | Taqiqlangan tashqi portlarga chiqishni cheklash |
| **P5** | LAN → WEB-DMZ | HTTP 80 | **Accept** | — | Ichki xodimlar DMZ dagi veb-sayt / portalni ko‘rishi |
| **P6** | LAN → WEB-DMZ | SSH 22 | **Drop** | — | Ushbu variantda DMZ serverini SSH orqali boshqarish yopiq |
| **P7** | WAN → FW-RTR:8080 | TCP 8080 | **Accept** | **DNAT** | Tashqi mijozlar uchun DMZ veb-serverini (port 80) e’lon qilish |
| **P8** | DMZ → LAN | Barcha yangi oqimlar | **Drop** | — | Agar DMZ buzilsa, xaker ichki LAN tarmog‘iga o‘tolmasligi |

---

## 5. MASHQNI BAJARISHNING BOSQICHMA-BOSQICH YO‘RIQNOMASI

### 0-BOSQICH: Ubuntu Hostda VM Nusxalarini Tayyorlash

Host Ubuntu terminalida ish bajarasiz.

#### 1. Kerakli vositalarni o‘rnatish:
Arxivni ochish uchun `p7zip-full` paketini o‘rnating:
```bash
sudo apt update && sudo apt install -y p7zip-full
```
*Tushuntirish:* `p7zip-full` — Linuxda `.7z` formatidagi siqilgan arxivlarni ochish uchun yuqori unumdorlikka ega ochiq kodli arxivator dasturi.

#### 2. Laboratoriya papkalarini yaratish va arxivni ochish:
```bash
mkdir -p ~/A4-LAB/{FW-RTR,LAN-CLIENT,WEB-DMZ,EXT-1,Report}
```
*Tushuntirish:* `mkdir -p` — ko‘rsatilgan papkalar ierarxiyasini xatosiz va bir vaqtda yaratadi. `Report` papkasi barcha yakuniy fayllar va hisobotlar saqlanadigan joy.

Ubuntu Server tayyor obrazini (`UbuntuServer_24.04_VM.7z`) ochib, fayllarni 4 ta papkaga to‘liq nusxalang:
```bash
7z x /yo‘l/faylga/UbuntuServer_24.04_VM.7z -o~/A4-LAB/FW-RTR/
cp -r ~/A4-LAB/FW-RTR/* ~/A4-LAB/LAN-CLIENT/
cp -r ~/A4-LAB/FW-RTR/* ~/A4-LAB/WEB-DMZ/
cp -r ~/A4-LAB/FW-RTR/* ~/A4-LAB/EXT-1/
```

#### 3. VMware da virtual mashinalarni ro‘yxatga olish:
- VMware Workstation-ni oching (`vmware &`).
- `File` -> `Open` orqali har bir papkadagi `.vmx` faylni alohida oching.
- VM sozlamalarida ularning nomlarini mos ravishda o‘zgartiring: `FW-RTR`, `LAN-CLIENT`, `WEB-DMZ`, `EXT-1`.
- **Birinchi marta ishga tushirishda (Power On) VMware so‘rov bersa:**
  > Qat’iy ravishda **"I Copied It"** tugmasini bosing!
  > *Sababi:* Shunda VMware har bir virtual mashinaga yangi va unikal tarmoq MAC manzilini generatsiya qiladi. Agar "I Moved It" tanlansa, hamma VMlarda MAC bir xil bo‘lib qolib, tarmoqda konflikt yuzaga keladi.

#### 4. Resurslarni ajratish (VM Settings):
- **FW-RTR:** 2 vCPU, 2 GB RAM, 3 ta tarmoq adapteri (A4-LAN, A4-DMZ, A4-WAN) + 1 ta vaqtinchalik NAT adapteri.
- **LAN-CLIENT:** 1 vCPU, 1.5 GB RAM, 1 ta adapter (A4-LAN) + 1 ta vaqtinchalik NAT.
- **WEB-DMZ:** 1 vCPU, 2 GB RAM, 1 ta adapter (A4-DMZ) + 1 ta vaqtinchalik NAT.
- **EXT-1:** 1 vCPU, 1.5 GB RAM, 1 ta adapter (A4-WAN) + 1 ta vaqtinchalik NAT.

---

### 1-BOSQICH: Lokal Wi-Fi Orqali Paketlarni O‘rnatish va Snapshot Olish

Har bir virtual mashinani navbat bilan yoqing va uning konsoliga kiring.

#### 1. Hostname belgilash:
Har bir VMda o‘z nomini yozing:
- **FW-RTR da:**
  ```bash
  sudo hostnamectl set-hostname FW-RTR
  ```
- **LAN-CLIENT da:**
  ```bash
  sudo hostnamectl set-hostname LAN-CLIENT
  ```
- **WEB-DMZ da:**
  ```bash
  sudo hostnamectl set-hostname WEB-DMZ
  ```
- **EXT-1 da:**
  ```bash
  sudo hostnamectl set-hostname EXT-1
  ```
*Tushuntirish:* `hostnamectl set-hostname <nom>` — operatsion tizimning tarmoqdagi nomini (hostname) o‘zgartiradi va `/etc/hostname` fayliga yozadi. Terminal promptida xatolik bo‘lmasligi uchun qayta kiring (`exec bash`).

#### 2. Paketlarni o‘rnatish (Shaxsiy Wi-Fi dan to‘g‘ridan-to‘g‘ri, proksisiz!):
Tizimda vaqtinchalik NAT adapteri orqali internet borligini tekshiring (`ping -c 2 8.8.8.8`). Hech qanday proksi sozlamang!

- **FW-RTR konsolida:**
  ```bash
  sudo apt update
  sudo apt install -y nftables conntrack tcpdump openssh-server iproute2
  sudo systemctl enable --now ssh
  ```
  *Tushuntirish:* `nftables` — asosiy xavfsizlik devori; `conntrack` — sessiyalarni ko‘rish utilitasi; `tcpdump` — paketlarni ushlash vositasi; `openssh-server` — SSH xizmati.

- **LAN-CLIENT konsolida:**
  ```bash
  sudo apt update
  sudo apt install -y curl netcat-openbsd iputils-ping tcpdump iproute2
  ```
  *Tushuntirish:* `curl` — HTTP so‘rovlar yuborish; `netcat-openbsd` (nc) — ixtiyoriy TCP/UDP portlarni tekshirish; `iputils-ping` — ICMP diagnostikasi.

- **WEB-DMZ konsolida:**
  ```bash
  sudo apt update
  sudo apt install -y nginx openssh-server curl iputils-ping tcpdump
  sudo systemctl enable --now nginx ssh
  ```
  *Tushuntirish:* `nginx` — DMZ veb-serveri sifatida ishlatiladigan dastur.

- **EXT-1 konsolida:**
  ```bash
  sudo apt update
  sudo apt install -y python3 curl netcat-openbsd tcpdump iproute2
  ```
  *Tushuntirish:* `python3` — sinov uchun 8000 va 9000 portlarda engil HTTP serverlar ko‘tarish uchun ishlatiladi.

#### 3. NAT adapterni uzish va Boshlang‘ich Snapshot olish:
Barcha VMlarni to‘xtating (`sudo poweroff`).
VMware sozlamalariga kirib:
1. Vaqtinchalik NAT adapterida `Connected` va `Connect at power on` belgilarini olib tashlang (faqat LAN segment adapterlari qolsin).
2. Har bir virtual mashinada snapshot oling:
   - VMware menyusi: `VM` -> `Snapshot` -> `Take Snapshot...`
   - Nomi: **`A4-base`**
   - Tavsifi: *Barcha paketlar o'rnatilgan, NAT uzilgan, tajribaga tayyor toza holat.*

---

### 2-BOSQICH: Virtual Interfeyslarni Aniqlash va FW-RTR Routing Sozlamalari

FW-RTR mashinasini yoqing va konsoliga kiring.

#### 1. Interfeys nomlarini aniqlash:
```bash
ip -br link
for i in /sys/class/net/en*; do echo "$i -> $(cat "$i/address")"; done
```
*Tushuntirish:* `ip -br link` — barcha tarmoq interfeyslarining qisqa holatini ko‘rsatadi. Sikl (`for`) har bir tarmoq interfeysining tizimdagi nomi va uning MAC manzilini chiqaradi. Chiqqan MAC manzillarni VMware Settings dagi tarmoq adapterlarining `Advanced` bo‘limidagi MAC manzillar bilan solishtirib, qaysi interfeys qaysi segmentga tegishli ekanini aniqlang!

Faraz qilaylik (ko‘p hollarda VMware da shunday bo‘ladi):
- `ens37` (yoki `enp0s8`) = **A4-LAN**
- `ens38` (yoki `enp0s9`) = **A4-DMZ**
- `ens39` (yoki `enp0s10`) = **A4-WAN**

#### 2. FW-RTR interfeyslariga statik IP manzillar biriktirish:
O‘z interfeys nomlaringizni o‘zgaruvchilarga yozing va buyruqlarni bajaring:
```bash
LAN_IF=ens37
DMZ_IF=ens38
WAN_IF=ens39

for IFACE in "$LAN_IF" "$DMZ_IF" "$WAN_IF"; do
    sudo ip addr flush dev "$IFACE"
    sudo ip link set "$IFACE" up
done

sudo ip addr add 10.40.10.1/24 dev "$LAN_IF"
sudo ip addr add 10.40.30.1/24 dev "$DMZ_IF"
sudo ip addr add 198.18.0.1/24 dev "$WAN_IF"
```
*Tushuntirish:*
- `ip addr flush dev "$IFACE"` — interfeysdagi barcha eski yoki DHCP orqali olingan tasodifiy IP manzillarni tozalaydi.
- `ip link set "$IFACE" up` — tarmoq interfeysini faol (UP) holatga keltiradi.
- `ip addr add <IP/mask> dev <IFACE>` — interfeysga belgilangan statik IP manzil va tarmoq niqobini biriktiradi.

#### 3. IPv4 marshrutlashni (Forwarding) yoqish:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-a4-router.conf
```
*Tushuntirish:*
- `sysctl -w net.ipv4.ip_forward=1` — Linux yadrosiga bir tarmoq interfeysidan kelgan paketni ikkinchi tarmoq interfeysiga tranzit o‘tkazishga (router bo‘lib ishlashga) ruxsat beradi.
- `tee /etc/sysctl.d/99-a4-router.conf` — bu sozlamani tizim qayta yuklanganda ham (reboot) saqlanib qoladigan qilib konfiguratsiyaga yozadi.

Tekshirish:
```bash
ip -br addr
sysctl net.ipv4.ip_forward
```

---

### 3-BOSQICH: LAN-CLIENT, WEB-DMZ va EXT-1 Mijozlarini Sozlash

Har bir mijoz VM konsolida uning interfeysini ko‘rib chiqing (`ip -br link`, odatda bitta adapter bo‘lgani uchun `ens33` bo‘ladi).

#### 1. LAN-CLIENT da:
```bash
LAB_IF=ens33
sudo ip addr flush dev "$LAB_IF"
sudo ip link set "$LAB_IF" up
sudo ip addr add 10.40.10.10/24 dev "$LAB_IF"
sudo ip route replace default via 10.40.10.1 dev "$LAB_IF"
ping -c 2 10.40.10.1
```
*Tushuntirish:*
- `ip addr add 10.40.10.10/24 ...` — LAN mijoziga `10.40.10.10` IP manzilini beradi.
- `ip route replace default via 10.40.10.1 ...` — standart shlyuz (default gateway) sifatida FW-RTR ning LAN manzilini ko‘rsatadi. Barcha boshqa tarmoqqa yo‘naltirilgan paketlar FW-RTR ga boradi.
- `ping -c 2 10.40.10.1` — shlyuz bilan ulanish borligini tekshiradi (2 ta ICMP paketi).

#### 2. WEB-DMZ da:
```bash
LAB_IF=ens33
sudo ip addr flush dev "$LAB_IF"
sudo ip link set "$LAB_IF" up
sudo ip addr add 10.40.30.20/24 dev "$LAB_IF"
sudo ip route replace default via 10.40.30.1 dev "$LAB_IF"
ping -c 2 10.40.30.1
```
*Tushuntirish:* WEB-DMZ serveriga `10.40.30.20` IP manzili va standart shlyuz sifatida FW-RTR ning DMZ interfeysi (`10.40.30.1`) ko‘rsatiladi.

#### 3. EXT-1 da:
```bash
LAB_IF=ens33
sudo ip addr flush dev "$LAB_IF"
sudo ip link set "$LAB_IF" up
sudo ip addr add 198.18.0.10/24 dev "$LAB_IF"
sudo ip route del default 2>/dev/null || true
ping -c 2 198.18.0.1
```
*Tushuntirish:* EXT-1 ga `198.18.0.10` IP manzili beriladi. `ip route del default` — undan default gateway o‘chiriladi. Nega? Chunki bu yopiq WAN muhitida EXT-1 private tarmoqlarni (`10.40.10.0/24`) o‘z marshrutlar jadvalida bilmasligi shart! EXT-1 faqat `198.18.0.1` ni ko‘radi, bu esa LAN dan chiquvchi trafikka SNAT nima uchun zarurligini isbotlaydi.

---

### 4-BOSQICH: WEB-DMZ va EXT-1 da Sinov Xizmatlarini Ishga Tushirish

#### 1. WEB-DMZ da Nginx sahifasini tayyorlash:
```bash
echo '<h1>A4 DMZ service</h1><p>10.40.30.20:80</p>' | sudo tee /var/www/html/index.html
sudo nginx -t
sudo systemctl reload nginx
sudo ss -lntp | grep -E ':(22|80)\b'
curl --noproxy "*" http://127.0.0.1/
```
*Tushuntirish:*
- `tee /var/www/html/index.html` — DMZ veb-serveri uchun laboratoriya matnini yozadi.
- `nginx -t` — Nginx konfiguratsiyasi sintaksisida xato yo‘qligini tekshiradi.
- `systemctl reload nginx` — Nginx xizmatini to‘xtatmasdan yangi sozlamalarni yuklaydi.
- `ss -lntp` — tizimda 22 (SSH) va 80 (HTTP) portlar tinglayotganini (Listening) ko‘rsatadi.
- `curl --noproxy "*"` — proksisiz lokal veb-sahifani chaqirib, uning ishlayotganini tasdiqlaydi.

#### 2. EXT-1 da Python sinov HTTP serverlarini ishga tushirish:
EXT-1 tashqi tarmoq (Internet) vazifasini bajaradi. Unda ikkita xizmat ko‘taramiz: 8000 (ruxsat beriladigan) va 9000 (taqiqlanadigan).
```bash
nohup python3 -m http.server 8000 --bind 198.18.0.10 > ~/A4_ext_8000.log 2>&1 &
nohup python3 -m http.server 9000 --bind 198.18.0.10 > ~/A4_ext_9000.log 2>&1 &
sleep 1
ss -lntp | grep -E ':(8000|9000)\b'
curl --noproxy "*" http://198.18.0.10:8000/ >/dev/null && echo "8000 OK"
curl --noproxy "*" http://198.18.0.10:9000/ >/dev/null && echo "9000 OK"
```
*Tushuntirish:*
- `nohup ... &` — jarayonni fon rejimida ishga tushiradi, terminal yopilsa ham u to‘xtab qolmaydi.
- `python3 -m http.server <port> --bind <IP>` — ko‘rsatilgan IP va portda tezkor HTTP veb-server ochadi.
- `ss -lntp` — ikkala port (8000 va 9000) ham haqiqatan ishlab turganini isbotlaydi. Bu juda muhim: keyinroq 9000 port ishlamay qolganda, bunga sabab xizmat yo‘qligi emas, aynan firewall bloklagani ekani isbot bo‘ladi.

---

### 5-BOSQICH: FW-RTR da nftables Qoidalarini (Ruleset) Yozish

FW-RTR konsoliga kiring. Barcha firewall qoidalarini `/etc/nftables.conf` fayliga yaxlit holda yozamiz.

Eski qoidalarni saqlab qo‘ying va yangi fayl yarating:
```bash
sudo nft list ruleset | tee ~/A4_ruleset_before.txt
sudo cp -a /etc/nftables.conf /etc/nftables.conf.a4-backup
```

Endi `/etc/nftables.conf` faylini yarating (interfeys nomlarini o‘z tizimingizdagi nomlarga moslang, masalan `ens37`, `ens38`, `ens39`):

```bash
sudo tee /etc/nftables.conf >/dev/null <<'EOF'
#!/usr/sbin/nft -f
flush ruleset

# Interfeyslar va tarmoq parametrlarini aniqlash (o'zgaruvchilar)
define LAN_IF = "ens37"
define DMZ_IF = "ens38"
define WAN_IF = "ens39"

define LAN_NET = 10.40.10.0/24
define DMZ_WEB = 10.40.30.20
define WAN_IP  = 198.18.0.1

# -------------------------------------------------------------
# 1. FILTER JADVALI (IPv4 va IPv6 uchun inet oilasi)
# -------------------------------------------------------------
table inet a4_filter {

    # INPUT ZANJIRI: Faqat firewallning o'ziga keluvchi trafik
    chain input {
        type filter hook input priority filter; policy drop;

        # Loopback interfeysiga to'liq ruxsat
        iifname "lo" counter accept

        # Noto'g'ri (anomal) paketlarni darhol tashlab yuborish
        ct state invalid counter drop

        # O'rnatilgan va bog'liq sessiyalarning javob paketlarini o'tkazish
        ct state established,related counter accept

        # P1: LAN tarmog'idan FW-RTR ga SSH boshqaruvi va Ping diagnostikasi
        iifname $LAN_IF ip saddr $LAN_NET tcp dport 22 counter accept
        iifname $LAN_IF ip saddr $LAN_NET icmp type echo-request counter accept

        # P2: WAN va DMZ dan yoki qolgan barcha ruxsatsiz kirishlarni jurnallash va bloklash
        limit rate 5/minute log prefix "A4-INPUT-DROP " flags all
        counter drop
    }

    # FORWARD ZANJIRI: Firewall orqali boshqa tarmoqqa o'tuvchi tranzit trafik
    chain forward {
        type filter hook forward priority filter; policy drop;

        # Noto'g'ri paketlarni tashlash
        ct state invalid counter drop

        # Stateful nazorat: ruxsat etilgan sessiyalarning javob oqimlari
        ct state established,related counter accept

        # P3: LAN dan WAN ga faqat TCP 8000 xizmatiga ruxsat
        iifname $LAN_IF oifname $WAN_IF ip saddr $LAN_NET tcp dport 8000 counter accept

        # P5: LAN dan DMZ dagi veb-serverga HTTP 80 ga ruxsat
        iifname $LAN_IF oifname $DMZ_IF ip saddr $LAN_NET ip daddr $DMZ_WEB tcp dport 80 counter accept

        # P7: WAN dan kelgan va DNAT qilingan trafikni DMZ veb-serveriga o'tkazish
        iifname $WAN_IF oifname $DMZ_IF ip daddr $DMZ_WEB tcp dport 80 ct status dnat counter accept

        # P4, P6, P8: Barcha qolgan tranzit urinishlarni (TCP 9000, SSH, DMZ->LAN) jurnallash va bloklash
        limit rate 5/minute log prefix "A4-FWD-DROP " flags all
        counter drop
    }

    # OUTPUT ZANJIRI: Firewallning o'zidan chiquvchi paketlar
    chain output {
        type filter hook output priority filter; policy accept;
    }
}

# -------------------------------------------------------------
# 2. NAT JADVALI (Faqat IPv4 uchun ip oilasi)
# -------------------------------------------------------------
table ip a4_nat {

    # PREROUTING: Paket marshrutlanishidan oldin manzilni tarjima qilish (DNAT)
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # P7: WAN:8080 portiga kelgan so'rovni DMZ veb-serveriga (10.40.30.20:80) yo'naltirish
        iifname $WAN_IF ip daddr $WAN_IP tcp dport 8080 counter dnat to 10.40.30.20:80
    }

    # POSTROUTING: Paket marshrutlanib, chiqib ketishidan oldin manzilni tarjima qilish (SNAT)
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        # P3: LAN tarmog'idan WAN interfeysi orqali chiquvchi paketlar manbasini 198.18.0.1 ga aylantirish
        oifname $WAN_IF ip saddr $LAN_NET counter snat to 198.18.0.1
    }
}
EOF
```

---

### 6-BOSQICH: Sintaksisni Tekshirish, Rulesetni Qo‘llash va Audit

#### 1. Sintaksis nazorati:
Qoidalarni birdaniga qo‘llamasdan, avval sintaksisini sinab ko‘ring:
```bash
sudo nft -c -f /etc/nftables.conf
echo "Sintaksis tekshiruv kodi: $?"
```
*Tushuntirish:* `nft -c -f <fayl>` — `-c` (check) kaliti qoidalarni yadroga yuklamasdan, sintaksisini tekshiradi. `$?` qiymati `0` chiqishi shart! Agar `0` chiqmasa, qaysi qatorda xato yozilgani terminalda ko‘rsatiladi.

#### 2. Rulesetni yadroga yuklash:
```bash
sudo nft -f /etc/nftables.conf
```
*Tushuntirish:* `nft -f <fayl>` — fayldagi barcha jadvallar, zanjirlar va qoidalarni to‘liq Linux yadrosiga yuklaydi (`flush ruleset` orqali eski qoidalarni avval tozalaydi).

#### 3. Audit va tekshiruv:
Qoidalar yuklanganini, zanjirlar `policy drop` ekanini va counterlar mavjudligini ko‘ring:
```bash
sudo nft -a list table inet a4_filter
sudo nft -a list table ip a4_nat
sudo nft list ruleset | tee ~/A4_ruleset_final.txt
sudo systemctl enable nftables
```
*Tushuntirish:* `-a` kaliti har bir qoidaning unikal raqamini (`handle <raqam>`) ko‘rsatadi. Bu handle keyinchalik alohida qoidani o‘chirish yoki tahlil qilish uchun zarur bo‘ladi.

---

### 7-BOSQICH: 10 ta Ijobiy va Salbiy Testlarni O‘tkazish (T1–T10)

Ushbu bosqichda siz har bir testni belgilangan VMda bajarasiz va uning natijasini FW-RTR da hisoblagichlar (`counter`) yoki loglar bilan bog‘laysiz.

#### 1. LAN-CLIENT VM da testlar:
```bash
# T1: LAN -> FW-RTR ICMP Ping (Kutiladi: 0% packet loss, MUVAFFAQISHLI)
ping -c 2 -W 2 10.40.10.1

# T2: LAN -> FW-RTR SSH 22 (Kutiladi: Connection to 10.40.10.1 22 port [tcp/ssh] succeeded!)
nc -vz -w 3 10.40.10.1 22

# T4: LAN -> EXT-1 TCP 8000 (Kutiladi: HTTP 200 OK, MUVAFFAQISHLI / SNAT orqali)
curl --noproxy "*" --max-time 5 -I http://198.18.0.10:8000/

# T5: LAN -> EXT-1 TCP 9000 (Kutiladi: Connection timed out / BLOKLANGAN)
curl --noproxy "*" --max-time 5 http://198.18.0.10:9000/

# T6: LAN -> WEB-DMZ HTTP 80 (Kutiladi: "A4 DMZ service" sahifasi chiqadi, MUVAFFAQISHLI)
curl --noproxy "*" --max-time 5 http://10.40.30.20/

# T7: LAN -> WEB-DMZ SSH 22 (Kutiladi: Connection timed out / BLOKLANGAN)
nc -vz -w 3 10.40.30.20 22
```

#### 2. EXT-1 VM da testlar:
```bash
# T3: EXT-1 -> FW-RTR SSH 22 (Kutiladi: Connection timed out / BLOKLANGAN)
nc -vz -w 3 198.18.0.1 22

# T8: EXT-1 -> FW-RTR:8080 (Kutiladi: DNAT orqali "A4 DMZ service" chiqadi, MUVAFFAQISHLI)
curl --noproxy "*" --max-time 5 http://198.18.0.1:8080/

# T9: EXT-1 -> FW-RTR:80 (Kutiladi: Connection timed out / DNAT faqat 8080 da, to'g'ridan-to'g'ri 80 BLOKLANGAN)
curl --noproxy "*" --max-time 5 http://198.18.0.1:80/
```

#### 3. WEB-DMZ VM da test:
```bash
# T10: WEB-DMZ -> LAN-CLIENT Ping (Kutiladi: 100% packet loss / BLOKLANGAN)
ping -c 2 -W 2 10.40.10.10
```

#### 4. FW-RTR da testlar natijalarini (Counter va Loglarni) tekshirish:
Har bir testdan so‘ng FW-RTR da quyidagi buyruqlarni bajaring:
```bash
sudo nft -a list chain inet a4_filter input
sudo nft -a list chain inet a4_filter forward
sudo nft -a list table ip a4_nat
sudo journalctl -k -n 40 --no-pager | grep 'A4-'
```
*Tushuntirish:* Qoidalar qarshisidagi `packets <son> bytes <son>` qiymatlari oshganini ko‘rasiz. Bloklangan testlar (`T3`, `T5`, `T7`, `T9`, `T10`) esa `journalctl` da `A4-INPUT-DROP` yoki `A4-FWD-DROP` yozuvlari bilan ko‘rinadi.

---

### 8-BOSQICH: Paket Darajasida Tahlil: tcpdump va conntrack Dalillari

#### 1. SNAT ning ishlashini isbotlash:
SNAT ishlaganda EXT-1 paketi `10.40.10.10` dan emas, `198.18.0.1` dan kelgandek ko‘rishi shart!

- **EXT-1 terminalida** tcpdump ni yoqing:
  ```bash
  sudo timeout 15 tcpdump -ni ens33 'tcp port 8000 and host 198.18.0.1' -c 6
  ```
- **Shu paytning o‘zida LAN-CLIENT terminalida** so‘rov yuboring:
  ```bash
  curl --noproxy "*" --max-time 5 http://198.18.0.10:8000/ >/dev/null
  ```
- **Natija:** EXT-1 dagi tcpdump quyidagicha qatorni ko‘rsatadi:
  `IP 198.18.0.1.xxxxx > 198.18.0.10.8000: Flags [S]...`
  Bu paket LAN-CLIENT dan chiqqan bo‘lsa ham, uning manba IP si FW-RTR tomonidan `198.18.0.1` ga aylantirilganini (SNAT) 100% isbotlaydi.

#### 2. DNAT ning ishlashini isbotlash:
DNAT da paket WAN interfeysiga kelganda `198.18.0.1:8080` bo‘ladi, lekin DMZ interfeysidan chiqqanda `10.40.30.20:80` ga aylanadi!

- **FW-RTR da ikkita alohida terminal oching:**
  - 1-terminal (WAN):
    ```bash
    sudo tcpdump -ni ens39 'host 198.18.0.10 and tcp port 8080'
    ```
  - 2-terminal (DMZ):
    ```bash
    sudo tcpdump -ni ens38 'host 10.40.30.20 and tcp port 80'
    ```
- **EXT-1 da so‘rov yuboring:**
  ```bash
  curl --noproxy "*" --max-time 5 http://198.18.0.1:8080/
  ```
- **Natija:** WAN interfeysida port 8080 bo‘lgan paket, DMZ interfeysida port 80 va IP `10.40.30.20` ga aylangan holda qayd etiladi.

#### 3. PCAP fayl va Conntrack holatini saqlash:
FW-RTR terminalida:
```bash
# Tarmoq oqimini 20 soniya PCAP faylga yozish
sudo timeout 20 tcpdump -Z root -ni any '(host 198.18.0.10 or host 10.40.30.20)' -w ~/A4_nat.pcap &

# Shu vaqtda test so'rovlarini bajaring (LAN dan EXT ga, EXT dan FW ga)
sleep 2
curl --noproxy "*" --max-time 3 http://198.18.0.1:8080/ 2>/dev/null || true

# Conntrack sessiyalar jadvalini matn faylga saqlash
sudo conntrack -L -p tcp | grep -E '10.40.10.10|10.40.30.20|198.18.0.10' | tee ~/A4_conntrack.txt
ls -lh ~/A4_nat.pcap ~/A4_conntrack.txt
```
*Tushuntirish:* `conntrack -L` buyrug‘i jadvaldagi ulanishlarni chiqaradi. Unda quyidagicha yozuv ko‘rinadi:
`src=10.40.10.10 dst=198.18.0.10 sport=... dport=8000 src=198.18.0.10 dst=198.18.0.1 sport=8000 dport=... [ASSURED]`
Bu conntrack har ikki yo‘nalishdagi IP/port tarjimalarini aniq eslab qolganini bildiradi.

---

### 9-BOSQICH: Nazoratli Xatolik: established,related Qoidasini O‘chirish va Tahlil

Bu tajriba stateful firewall nimaga kerakligini amalda isbotlaydi.

#### 1. Forward zanjiridagi `established,related` qoidasining handle raqamini toping:
```bash
sudo nft -a list chain inet a4_filter forward
```
Quyidagiga o‘xshash qatorni ko‘rasiz:
`ct state established,related counter accept # handle 5`

Avtomatik topish va o‘chirish:
```bash
HANDLE=$(sudo nft -a list chain inet a4_filter forward | awk '/ct state established,related/{print $NF; exit}')
echo "O'chiriladigan qoida handle raqami: $HANDLE"
sudo nft delete rule inet a4_filter forward handle "$HANDLE"
```
*Tushuntirish:* `nft delete rule <jadval> <zanjir> handle <raqam>` — qoidani uning yagona identifikatori (handle) orqali o‘chirib tashlaydi.

#### 2. Xatolik alomatini kuzatish:
LAN-CLIENT terminalida ulanishni tekshiring:
```bash
curl --noproxy "*" --no-keepalive --max-time 5 http://198.18.0.10:8000/
echo "curl chiqish kodi: $?"
```
*Kuzatuv va ilmiy tushuntirish:*
- Buyruq uzoq o‘ylanib turadi va `curl: (28) Connection timed out after 5000 milliseconds` xatosi bilan (chiqish kodi `28`) tugaydi.
- **Sababi:** LAN-CLIENT dan chiquvchi birinchi `TCP SYN` paketi P3 ruxsat qoidasiga tushadi va EXT-1 ga yetib boradi. EXT-1 unga javoban `TCP SYN-ACK` qaytaradi. Lekin bu qaytuvchi javob paketi FW-RTR ga kelganda, uni o‘tkazib yuboruvchi `established,related` qoidasi endi yo‘q!
- Forward zanjirining oxiridagi `policy drop` ushbu qaytuvchi paketni tashlab yuboradi (`A4-FWD-DROP`). Natijada TCP uch bosqichli handshake (Handshake) yakunlanmaydi va sessiya uziladi.

#### 3. Qoidani xavfsiz tiklash:
Qoidani qayta tiklashning eng xavfsiz va standart yo‘li — oldindan tekshirilgan konfiguratsiya faylini qayta yuklashdir:
```bash
sudo nft -c -f /etc/nftables.conf && sudo nft -f /etc/nftables.conf
sudo nft -a list chain inet a4_filter forward
```
Endi LAN-CLIENT da qayta sinab ko‘ring:
```bash
curl --noproxy "*" --no-keepalive --max-time 5 http://198.18.0.10:8000/
```
Ulanish darhol muvaffaqiyatli ishlaydi!

---

### 10-BOSQICH: Dalillarni Saqlash va Tizimni Tozalash

FW-RTR da barcha dalil fayllarini tayyorlang:
```bash
sudo nft list ruleset | tee ~/A4_ruleset_final.txt
sudo cp /etc/nftables.conf ~/A4_nftables.conf
sudo nft -c -f ~/A4_nftables.conf
ls -lh ~/A4_ruleset_final.txt ~/A4_nftables.conf ~/A4_nat.pcap ~/A4_conntrack.txt
```

Bu fayllarni host Ubuntu mashinasiga nusxalash uchun VMware Shared Folders (`/mnt/a4share`) yoki oddiy `scp` dan foydalanishingiz mumkin.

Laboratoriya yakunlangach, FW-RTR ni dastlabki zaxira holatiga qaytarish:
```bash
sudo cp -a /etc/nftables.conf.a4-backup /etc/nftables.conf
sudo nft -c -f /etc/nftables.conf && sudo nft -f /etc/nftables.conf
sudo rm -f /etc/sysctl.d/99-a4-router.conf
sudo sysctl --system >/dev/null
```

---

## 6. HAR BIR CLI BUYRUQNING MUKAMMAL TAHLILI

Laboratoriyada ishlatilgan barcha buyruqlar va ularning kalitlari (flaglari):

### 1. Tarmoq va IP boshqaruvi (`iproute2`):
- `ip -br link`:
  - `-br` (`--brief`): Tarmoq interfeyslari (nomi, holati UP/DOWN, MAC manzili) haqida qisqa va qulay jadval ko‘rinishida ma’lumot beradi.
  - `link`: 2-darajali (Data Link layer) interfeyslar ro‘yxatini chiqaradi.
- `ip addr flush dev ens37`:
  - `addr flush`: Ko‘rsatilgan qurilmadagi barcha biriktirilgan IPv4 va IPv6 manzillarni bir zumda tozalaydi (DHCP dan qolib ketgan eskilarni yo‘qotadi).
  - `dev <nom>`: Aniq qaysi qurilma (device) ekanligini ko‘rsatadi.
- `ip link set ens37 up`:
  - `set ... up`: Tarmoq kartasini dasturiy jihatdan faollashtiradi (linkni yoqadi).
- `ip addr add 10.40.10.1/24 dev ens37`:
  - `addr add`: Interfeysga yangi statik IP manzil va tarmoq prefiksini (`/24` = `255.255.255.0`) biriktiradi.
- `ip route replace default via 10.40.10.1 dev ens33`:
  - `route replace`: Agar avvaldan default route mavjud bo‘lsa, uni xatosiz yangisiga almashtiradi.
  - `default via <IP>`: Barcha noma’lum manzillarga yo‘naltirilgan paketlar shlyuz (gateway) sifatida ushbu IP ga jo‘natilishini belgilaydi.
- `ip route get 10.40.10.10`:
  - Paket ushbu manzilga jo‘natilsa, yadro uni qaysi interfeys va qaysi IP orqali chiqarishini (marshrutlash qarorini) ko‘rsatadi.

### 2. Yadro parametrlari (`sysctl`):
- `sysctl -w net.ipv4.ip_forward=1`:
  - `-w`: Yadro parametriga yangi qiymat yozadi (`write`).
  - `net.ipv4.ip_forward=1`: Linux yadrosida IPv4 paketlarni marshrutlash (interfeyslararo uzatish) imkoniyatini yoqadi (`0` — o‘chiq, `1` — yoqiq).

### 3. Xavfsizlik devori (`nftables`):
- `nft -c -f /etc/nftables.conf`:
  - `-c` (`--check`): Faylni yadroga yuklamasdan faqat sintaksis tekshiruvini o‘tkazadi.
  - `-f` (`--file`): Qoidalar yozilgan konfiguratsiya faylini ko‘rsatadi.
- `nft -a list table inet a4_filter`:
  - `-a` (`--handle`): Har bir qoidaning unikal identifikatorini (`handle`) ko‘rsatadi (qoidalarni o‘chirish uchun zarur).
  - `list table <oila> <nom>`: Aniq bir jadvaldagi qoidalar ro‘yxatini chiqaradi.
- `nft delete rule inet a4_filter forward handle 5`:
  - `delete rule`: Belgilangan zanjirdan aynan ko‘rsatilgan `handle` raqamiga ega qoidani o‘chirib tashlaydi.
- `flush ruleset`:
  - Barcha jadvallar, zanjirlar va qoidalarni tozalab, yadroni bo‘sh holatga keltiradi.

### 4. Tarmoq diagnostikasi va tahlili (`tcpdump`, `conntrack`, `ss`, `curl`, `nc`):
- `tcpdump -ni ens39 'tcp port 8080' -c 6`:
  - `-n`: IP va portlarni domen nomlariga aylantirmaydi (DNS so‘rovlar bilan ushlab turmaydi, tez ishlaydi).
  - `-i ens39`: Aynan qaysi interfeysdagi paketlarni ushlashni ko‘rsatadi.
  - `'...'`: BPF (Berkeley Packet Filter) sintaksisi — faqat filtrga tushgan paketlar ushlanadi.
  - `-c 6`: Belgilangan sondagi paket ushlangach avtomatik to‘xtaydi.
  - `-w fayl.pcap`: Paketlarni Wiresharkda ko‘rish uchun xom formatda faylga yozadi.
- `conntrack -L -p tcp`:
  - `-L` (`--dump`): Holatlar jadvalidagi (connection tracking table) ulanishlar ro‘yxatini ko‘rsatadi.
  - `-p tcp`: Faqat TCP protokoli sessiyalarini filtrlaydi.
- `ss -lntp`:
  - `-l` (`--listening`): Faqat ulanishlarni kutayotgan (tinglayotgan) portlarni ko‘rsatadi.
  - `-n` (`--numeric`): Xizmat nomlarini emas, raqamli portlarni ko‘rsatadi.
  - `-t` (`--tcp`): Faqat TCP portlar.
  - `-p` (`--processes`): Portni qaysi dastur (PID/Process name) egallab turganini ko‘rsatadi.
- `curl --noproxy "*" --max-time 5 -I http://...`:
  - `--noproxy "*"`: Muhit o‘zgaruvchilaridagi har qanday proksi sozlamasini chetlab o‘tib, to‘g‘ridan-to‘g‘ri ulanadi.
  - `--max-time 5`: Agar server 5 soniya ichida javob bermasa, kutishni to‘xtatib chiqadi (bloklangan portlarni tekshirishda osilib qolmaslik uchun).
  - `-I` (`--head`): Faqat HTTP sarlavhalarini (Header) so‘raydi.
- `nc -vz -w 3 10.40.10.1 22`:
  - `-v` (`--verbose`): Batafsil ma’lumot beradi (ulanish muvaffaqiyatli yoki yo‘qligi).
  - `-z` (`--zero-I/O`): Ma’lumot uzatmaydi, faqat port ochiqligini (TCP handshake) tekshirib darhol chiqadi.
  - `-w 3`: Kutish vaqti (timeout) 3 soniya.

---

## 7. TO‘LDIRILADIGAN NATIJA JADVALLARI VA HISOBOT NAMUNASI

Talaba topshiradigan hisobotda ushbu jadvallar to‘liq aks etishi shart:

### 3.1. Interfeys va Adreslash Rejasi Jadvali

| VM nomi | Segment | Interfeys / MAC | IP / Prefiks | Gateway | Dalil ID |
| :--- | :--- | :--- | :--- | :--- | :---: |
| **FW-RTR** | A4-LAN | `ens37` (00:0c:29:xx:xx:01) | `10.40.10.1/24` | — | E1 |
| **FW-RTR** | A4-DMZ | `ens38` (00:0c:29:xx:xx:02) | `10.40.30.1/24` | — | E1 |
| **FW-RTR** | A4-WAN | `ens39` (00:0c:29:xx:xx:03) | `198.18.0.1/24` | — | E1 |
| **LAN-CLIENT** | A4-LAN | `ens33` (00:0c:29:xx:xx:10) | `10.40.10.10/24`| `10.40.10.1` | E2 |
| **WEB-DMZ** | A4-DMZ | `ens33` (00:0c:29:xx:xx:20) | `10.40.30.20/24`| `10.40.30.1` | E3 |
| **EXT-1** | A4-WAN | `ens33` (00:0c:29:xx:xx:30) | `198.18.0.10/24`| — (Yo‘q) | E4 |

---

### 3.2. Talabdan nftables Qoidasiga Izchillik Matritsasi

| P-ID | Manba / Maqsad | Chain / Hook | Match shartlari | Verdict / NAT | Handle / Dalil |
| :---: | :--- | :--- | :--- | :---: | :---: |
| **P1** | LAN → FW-RTR | `input` (filter) | `iifname $LAN_IF ip saddr $LAN_NET tcp dport 22` | **accept** | handle 6 |
| **P1** | LAN → FW-RTR | `input` (filter) | `iifname $LAN_IF ip saddr $LAN_NET icmp echo-request` | **accept** | handle 7 |
| **P2** | WAN/DMZ → FW | `input` (filter) | `policy drop` & `limit rate 5/m log prefix "A4-INPUT-DROP "` | **drop** | handle 8 |
| **P3** | LAN → WAN | `forward` (filter) | `iifname $LAN_IF oifname $WAN_IF ip saddr $LAN_NET tcp dport 8000`| **accept** | handle 6 |
| **P3** | LAN → WAN | `postrouting` (nat)| `oifname $WAN_IF ip saddr $LAN_NET` | **snat to 198.18.0.1** | handle 4 |
| **P4** | LAN → WAN (9000) | `forward` (filter) | Mos ruxsat yo‘q → zanjir oxiridagi log va `policy drop` | **drop** | handle 10 |
| **P5** | LAN → WEB-DMZ | `forward` (filter) | `iifname $LAN_IF oifname $DMZ_IF ip saddr $LAN_NET ip daddr $DMZ_WEB tcp dport 80` | **accept** | handle 7 |
| **P6** | LAN → DMZ (22) | `forward` (filter) | Mos ruxsat yo‘q → `policy drop` | **drop** | handle 10 |
| **P7** | WAN → FW:8080 | `prerouting` (nat) | `iifname $WAN_IF ip daddr $WAN_IP tcp dport 8080` | **dnat to 10.40.30.20:80** | handle 4 |
| **P7** | WAN → DMZ (80) | `forward` (filter) | `iifname $WAN_IF oifname $DMZ_IF ip daddr $DMZ_WEB tcp dport 80 ct status dnat` | **accept** | handle 8 |
| **P8** | DMZ → LAN | `forward` (filter) | Yangi ulanishga ruxsat yo‘q → `policy drop` | **drop** | handle 10 |

---

### 3.3. Trafik Testlari va Natijalar Jadvali (T1–T10)

| Test ID | Manba → Maqsad | Xizmat / Port | Kutilgan natija | Haqiqiy natija | P-ID / Counter / Log | Holati |
| :---: | :--- | :--- | :---: | :---: | :---: | :---: |
| **T1** | LAN → FW-RTR | ICMP Echo | Ruxsat | 0% packet loss | P1 / counter packets 2 | PASS |
| **T2** | LAN → FW-RTR | TCP 22 (SSH) | Ruxsat | Open / Succeeded | P1 / counter packets 2 | PASS |
| **T3** | EXT-1 → FW-RTR | TCP 22 (SSH) | Rad | Timed out | P2 / `A4-INPUT-DROP` log | PASS |
| **T4** | LAN → EXT-1 | TCP 8000 | Ruxsat + SNAT | HTTP 200 OK | P3 / counter packets 10 / SNAT capture | PASS |
| **T5** | LAN → EXT-1 | TCP 9000 | Rad | Timed out | P4 / `A4-FWD-DROP` log | PASS |
| **T6** | LAN → WEB-DMZ | TCP 80 (HTTP) | Ruxsat | HTTP 200 OK | P5 / counter packets 8 | PASS |
| **T7** | LAN → WEB-DMZ | TCP 22 (SSH) | Rad | Timed out | P6 / `A4-FWD-DROP` log | PASS |
| **T8** | EXT-1 → FW:8080| TCP 8080 (DNAT) | Ruxsat | "A4 DMZ service" | P7 / dnat counter 1 / forward counter | PASS |
| **T9** | EXT-1 → FW-RTR | TCP 80 | Rad | Timed out | P2 / `A4-INPUT-DROP` log | PASS |
| **T10**| WEB-DMZ → LAN | ICMP Ping | Rad | 100% packet loss | P8 / `A4-FWD-DROP` log | PASS |

---

### 3.4. NAT va Conntrack Kuzatuvlari Jadvali

| Dalil | Kuzatuv nuqtasi | NAT dan oldingi juftlik | NAT dan keyingi juftlik | Tahliliy izoh |
| :---: | :--- | :--- | :--- | :--- |
| **N1 (SNAT)** | FW-RTR: LAN kirishi vs WAN chiqishi | `10.40.10.10:48234` → `198.18.0.10:8000` | `198.18.0.1:48234` → `198.18.0.10:8000` | Manba IP private manzildan router WAN IP siga o‘zgardi. |
| **N2 (DNAT)** | FW-RTR: WAN kirishi vs DMZ chiqishi | `198.18.0.10:51240` → `198.18.0.1:8080` | `198.18.0.10:51240` → `10.40.30.20:80` | Maqsad IP va porti DMZ serverining ichki portiga o‘zgartirildi. |
| **N3 (Conntrack)**| FW-RTR: `conntrack -L` xotirasi | Asl oqim: `src=10.40.10.10 dst=198.18.0.10` | Kutilgan javob: `src=198.18.0.10 dst=198.18.0.1` | Yadro javob paketi qaytganda teskari tarjima qilishni saqlab qoldi. |
| **N4 (NAT Counter)**| FW-RTR: `a4_nat` jadvali | Yangi oqim (1 ta TCP SYN) | Counter +1 ga oshadi | NAT qoidasi oqimning faqat birinchi paketida ishlaydi. |

---

### 3.5. Nazoratli Xatolik va Tiklash Jurnali

| Bosqich | Alomat (Simptom) | Tekshiruv buyrug‘i | Aniqlangan sabab | Tiklash usuli | Yakuniy natija |
| :--- | :--- | :--- | :--- | :--- | :---: |
| **Bazaviy holat** | TCP 8000 ulanishi muvaffaqiyatli | `curl http://198.18.0.10:8000/` | Barcha qoidalar joyida | — | HTTP 200 OK |
| **Qoida olib tashlandi** | Ulanish muzlab qoladi va uziladi | `curl ...` -> exit code 28 | Forward zanjirida qaytuvchi `SYN-ACK` uchun ruxsat yo‘qolgan | — | Timed out |
| **Ruleset qayta yuklandi** | Ulanish zudlik bilan tiklandi | `sudo nft -f /etc/nftables.conf` | `ct state established,related` qoidasi qaytarildi | Konfiguratsiyadan qayta yuklash | HTTP 200 OK |

---

### 8.1. Talabaning 6–8 Gaplik Yakuniy Xulosasi (Namuna)
1. Mazkur laboratoriya mashg‘ulotida Ubuntu Server 24.04 negizida Netfilter/nftables vositasi yordamida uchta izolyatsiyalangan segment orasida xavfsiz marshrutizator va stateful firewall to‘liq sozlandi.
2. Xavfsizlik siyosati "Default Deny" tamoyiliga asoslanib, input va forward zanjirlarida barcha ruxsatsiz trafik qat’iy taqiqlandi va loglar rate-limiting bilan qayd etildi.
3. Ichki LAN segmentining tashqi WAN ga xavfsiz chiqishi uchun `postrouting` zanjirida statik SNAT sozlanib, EXT-1 tomonda manba IP `198.18.0.1` sifatida muvaffaqiyatli dalillandi.
4. Tashqi foydalanuvchilar DMZ veb-xizmatidan foydalanishi uchun `prerouting` zanjirida DNAT (8080 → 80) va unga mos keluvchi forward ruxsat qoidasi joriy etildi.
5. tcpdump va conntrack tahlillari orqali NAT faqat birinchi paketda ulanish holatini yaratishi, keyingi paketlar esa stateful jadval orqali avtomatik qayta ishlanishi isbotlandi.
6. Nazoratli xatolik tajribasida `established,related` qoidasi olib tashlanganda TCP uch bosqichli handshake ning qaytuvchi `SYN-ACK` paketi forward da bloklanishi amalda ko‘rsatildi.
7. Xulosa qilib aytganda, tarmoq marshrutlashi (routing), paketlarni filtrlash (firewalling) va manzil tarjimasi (NAT) bir-biri bilan uzviy bog‘liq, ammo Netfilter arxitekturasida alohida hook va zanjirlarda bajariluvchi mustaqil qatlamlar ekanligi to‘liq o‘rganildi.

---

## 8. O‘QITUVCHI UCHUN HIMOYA SAVOLLARI VA MUKAMMAL JAVOBLAR

Topshirish paytida o‘qituvchi berishi mumkin bo‘lgan barcha 14 ta savolga tayyor professional javoblar:

#### 1. Firewallning input va forward zanjirlari qaysi trafikni qayta ishlaydi?
**Javob:** `input` zanjiri faqat firewallning o‘ziga (uning lokal IP manzillariga) yo‘naltirilgan trafikni (masalan, FW-RTR ning SSH boshqaruvi yoki uning o‘ziga berilgan ping) qayta ishlaydi. `forward` zanjiri esa firewall orqali bir tarmoq segmentidan boshqa tarmoq segmentiga tranzit o‘tib ketayotgan paketlarni (masalan, LAN dan WAN ga yoki WAN dan DMZ ga) filtrlaydi.

#### 2. Stateful firewall paketlar tarixini qaysi mexanizm orqali hisobga oladi?
**Javob:** Linux yadrosining `conntrack` (Connection Tracking) moduli orqali. U xotirada jadval yuritib, har bir ulanishning 5 talik parametrini (Protokol, Manba IP, Manba Port, Maqsad IP, Maqsad Port) va uning holatini (`NEW`, `ESTABLISHED`, `RELATED`, `INVALID`) doimiy kuzatib boradi.

#### 3. Nega established,related qoidasi yangi WAN ulanishiga ruxsat bermaydi?
**Javob:** Chunki tashqaridan kelayotgan har qanday yangi ulanish paketi (TCP SYN) `conntrack` holatlar jadvalida hali mavjud bo‘lmaydi va u `NEW` holatida bo‘ladi. `established,related` esa faqat ichkaridan avval boshlangan va javobi kutilayotgan ulanishlargagina mos keladi.

#### 4. Default deny siyosati qoidalar ro‘yxatida qanday afzallik beradi?
**Javob:** Eng kam vakolat (Least Privilege) tamoyilini ta’minlaydi. Administrator qoidalar ro‘yxatida xatoga yo‘l qo‘ysa yoki yangi noma’lum xizmat paydo bo‘lsa ham, u avtomatik ravishda bloklanadi. Bu tizimni "Zero-Day" hujumlar va ochiq qolgan xavfli portlardan himoya qiladi.

#### 5. SNAT va DNAT da paketning qaysi manzil maydoni o‘zgaradi?
**Javob:** SNAT (Source NAT) da paket sarlavhasidagi **Manba (Source) IP manzili** (va zarurat tug‘ilganda manba porti) o‘zgartiriladi. DNAT (Destination NAT) da esa paket sarlavhasidagi **Maqsad (Destination) IP manzili va Maqsad porti** o‘zgartiriladi.

#### 6. Nega ushbu variantda dynamic WAN uchun masquerade emas, statik manzil uchun SNAT tanlandi?
**Javob:** `masquerade` qoidasi har bir yangi paket uchun tarmoq interfeysining IP manzilini qayta so‘rab oladi (DHCP orqali o‘zgaruvchan dinamik IP lar uchun mo‘ljallangan) va bu protsessorga ortiqcha yuklama beradi. Ushbu laboratoriyada WAN interfeysi statik IP (`198.18.0.1`) ga ega bo‘lgani sababli, to‘g‘ridan-to‘g‘ri `snat to 198.18.0.1` ko‘rsatish ancha tezkor va optimal hisoblanadi.

#### 7. DNAT qoidasi mavjud bo‘lsa ham forward ruxsatisiz tashqi HTTP nima uchun ishlamaydi?
**Javob:** Chunki Netfilter arxitekturasida NAT va Filter alohida vazifani bajaradi. DNAT `prerouting` bosqichida paketning manzilini o‘zgartiradi, ammo uni o‘tkazib yubormaydi. Shundan so‘ng paket marshrutlash orqali `forward` zanjiriga kiradi. Agar `forward` zanjirida `policy drop` bo‘lsa va paketga aniq ruxsat qoidasi (`ct status dnat accept`) bo‘lmasa, firewall uni xavfli paket deb hisoblab tashlab yuboradi.

#### 8. EXT-1 da 8000 va 9000 portlarining ikkalasi tinglayotganini isbotlash nima uchun kerak?
**Javob:** Agar 9000-portda xizmat ishlamayotgan bo‘lsa, LAN dan unga ulanish muvaffaqiyatsiz bo‘lishi tabiiy bo‘lardi (RST qaytardi). Testning maqsadi esa — xizmat serverda ishlab turgan bo‘lsa ham, aynan FW-RTR xavfsizlik devori uni bloklaganini (`A4-FWD-DROP`) isbotlashdan iborat.

#### 9. NAT counteri nega capture dagi paketlar soniga teng bo‘lmasligi mumkin?
**Javob:** Chunki NAT jadvallari (`prerouting` va `postrouting`) faqatgina ulanishning birinchi paketi (`NEW` holatidagi paket) kelgandagina chaqiriladi. Manzil tarjimasi o‘rnatilgach, keyingi barcha paketlar to‘g‘ridan-to‘g‘ri `conntrack` xotirasi orqali avtomatik tarjima qilinadi va NAT counteri har bir keyingi paket uchun oshmaydi. Capture da esa sessiyaning barcha (SYN, ACK, DATA, FIN) paketlari hisoblanadi.

#### 10. T4 testida EXT-1 manba sifatida qaysi IPni ko‘radi va bu nimani isbotlaydi?
**Javob:** EXT-1 manba sifatida `10.40.10.10` ni emas, balki `198.18.0.1` ni ko‘radi. Bu FW-RTR routeri LAN dan chiquvchi private IP manzilini o‘zining tashqi WAN IP siga muvaffaqiyatli almashtirganini (SNAT ishlaganini) isbotlaydi.

#### 11. T8 oqimida 198.18.0.1:8080 qaysi manzil-port juftligiga aylantirildi?
**Javob:** `10.40.30.20:80` juftligiga (DMZ dagi veb-serverning ichki IP si va Nginx portiga) aylantirildi.

#### 12. established,related qoidasi olib tashlanganda TCP uch bosqichli handshake qayerda buziladi?
**Javob:** Handshake ning ikkinchi bosqichida buziladi. Mijoz yuborgan `SYN` paketi serverga yetib boradi, lekin server javob tariqasida yuborgan `SYN-ACK` paketi qaytib kelayotganda, FW-RTR ning `forward` zanjirida javob paketiga ruxsat yo‘qligi sababli `policy drop` ga uchraydi va tashlab yuboriladi. Natijada mijoz ulanishni o‘rnata olmaydi (Timed out).

#### 13. Log rate limit nima uchun kerak va u drop qarorini bekor qiladimi?
**Javob:** Rate limit tizim jurnallarini (loglarni) DoS hujumlaridan va disk to‘lib qolishidan himoya qilish uchun kerak. U drop qarorini aslo bekor qilmaydi: belgilangan me’yordan (masalan, 5/minute) ortiqcha kelgan bloklangan paketlar shunchaki logga yozilmaydi, ammo baribir qat’iy `drop` qilinadi.

#### 14. Ushbu laboratoriya rulesetini ishlab chiqarish (production) muhitiga ko‘chirishdan oldin yana nimalar baholanadi?
**Javob:** 
1. Kiberhujumlardan himoya (TCP SYN Flood ga qarshi `synproxy` yoki limitlar).
2. Xavfsiz masofaviy boshqaruv (SSH portini o‘zgartirish, SSH kalitlari orqali kirish, brute-force ga qarshi `fail2ban`).
3. IP Spoofing dan himoya (`rp_filter` yoki `fib saddr . iif oif missing drop` qoidalari).
4. MTU va MSS clamping (VPN yoki tunnel orqali paketlar bo‘linib ketmasligi uchun `tcp flags syn tcp mss set rt mtu`).
5. Yuqori yuklamalarda conntrack jadvali hajmini (`nf_conntrack_max`) oshirish.

---

## 9. FOYDALANILGAN RASMIY MANBALAR

1. **Ubuntu Server Rasmiy Hujjatlari:** Firewalls and nftables  
   https://documentation.ubuntu.com/server/how-to/security/firewalls/
2. **Netfilter Loyihasi Rasmiy Vikisi:** nftables HOWTO and Documentation  
   https://wiki.nftables.org/wiki-nftables/
3. **Linux nft(8) Manpage:** Ubuntu Noble Manpages  
   https://manpages.ubuntu.com/manpages/noble/en/man8/nft.8.html
4. **Linux conntrack(8) Tools:** Netfilter Connection Tracking User-space Guide  
   https://manpages.ubuntu.com/manpages/noble/en/man8/conntrack.8.html
5. **RFC 2544:** Benchmarking Methodology for Network Interconnect Devices  
   https://www.rfc-editor.org/rfc/rfc2544
6. **RFC 1918:** Address Allocation for Private Internets  
   https://www.rfc-editor.org/rfc/rfc1918
7. **RFC 5737:** IPv4 Address Blocks Reserved for Documentation  
   https://www.rfc-editor.org/rfc/rfc5737
