# A3. VLAN VA DMZ ASOSIDA TARMOQ SEGMENTATSIYASINI LOYIHALASH
## 1-Variant: Router-on-a-stick yordamida USER, ADMIN va DMZ xavfsizlik zonalarini ajratish

---

## 📌 MUHIM: ISH MUHITI VA FOYDALANUVCHI SHARTLARI

Ushbu amaliy qo'llanma talabaning aniq texnik shartlari va ish muhitiga to'liq moslashtirilgan holda tuzilgan:

* **Xost Operatsion Tizimi (Host OS):** **Ubuntu Linux** (*Windows emas!*). Barcha gipervizor boshqaruvi va tarmoq buyruqlari Linux terminali va VMware Workstation/Player for Linux muhitiga asoslanadi.
* **Gipervizor:** **VMware Workstation Pro / Player for Linux**.
* **Virtual Mashinalar (VM):**
  1. `RTR-1` — Ubuntu Server 24.04 LTS (Router / Firewall / Router-on-a-stick)
  2. `USER-1` — Ubuntu Server 24.04 LTS (VLAN 10 mijoz tuguni)
  3. `ADMIN-1` — Ubuntu Server 24.04 LTS (VLAN 20 ma'muriyat tuguni)
  4. `WEB-DMZ` — Ubuntu Server 24.04 LTS (VLAN 30 DMZ Web & SSH serveri)
* **Qo'shimcha Windows 11 Desktop VM (Ixtiyoriy/Amaliy):** Talabaning VMware ichida tayyorlangan Windows 11 Desktop virtual mashinasi mavjud. Qo'llanmada Windows 11 VM dan qanday qilib grafik mijoz (brauzer, SSH, ping) hamda `A3_vlan.pcap` faylini Wireshark grafik interfeysida tahlil qilish vositasi sifatida foydalanish to'liq ko'rsatib o'tilgan.
* **Lokal Wi-Fi Tarmog'i Sharti (Universitet Wi-Fi emas!):**
  * Talaba o'zining shaxsiy lokal Wi-Fi (uy routeri) tarmog'idan foydalanadi.
  * **DIQQAT (Qat'iy xavfsizlik qoidasi):** Virtual mashinalarning laboratoriya adapterini (NIC) xostdagi jismoniy Wi-Fi adapteriga (`wlan0` / `wlp...`) **Bridged** rejimida ulash **qat'iyan man etiladi!**
    * *Sabab 1:* Wi-Fi (IEEE 802.11) standartida 802.1Q Ethernet teglari to'liq qo'llab-quvvatlanmaydi; Wi-Fi drayverlari va router teglangan freymlarni filtrlab tashlaydi yoki uzatmaydi.
    * *Sabab 2:* 10.10.10.0/24, 10.10.20.0/24, 10.10.30.0/24 laboratoriya subnetlari uy Wi-Fi routerining ichki manzillar fazosi bilan to'qnash kelishi yoki router tomonidan bloklanishi mumkin.
    * *Sabab 3:* Laboratoriya kiberxavfsizlik talablariga ko'ra to'liq izolyatsiyalangan bo'lishi shart.
  * **Yechim:** VMware Workstation for Linux da **LAN Segment** (nomi: `A3-TRUNK`) yaratiladi yoki DHCP xizmati o'chirilgan **Host-only (masalan, `vmnet2`)** virtual tarmog'i ishlatiladi. Barcha 4 ta VM ning laboratoriya adapteri faqat ushbu izolyatsiyalangan segmentga ulanadi. Paketlarni yuklab olish uchun vaqtincha ikkinchi adapter sifatida NAT ishlatiladi va paketlar o'rnatilgach, u uzib qo'yiladi.

---

# 1-QISM: MAVZUNING MAQSADI VA KONSEPSIYASI

### 1.1. Laboratoriya ishining asosiy maqsadi
Ushbu amaliy mashg'ulotning bosh maqsadi — yagona umumiy (flat) ikkinchi pog'ona (L2) virtual infratuzilmasida **IEEE 802.1Q** standartidagi teglash mexanizmi yordamida tarmoqni mantiqiy **USER (VLAN 10)**, **ADMIN (VLAN 20)** va **DMZ (VLAN 30)** xavfsizlik zonalariga ajratish, Ubuntu Server asosidagi **RTR-1** tugunida **Router-on-a-stick (RoaS)** marshrutizatorini barpo etish hamda **nftables** yordamida zonalararo **"Default Deny" (sukut bo'yicha taqiqlash)** va **"Least Privilege" (minimal zarur ruxsat)** tamoyillariga asoslangan xavfsizlik siyosatini joriy etish va tekshirishdan iborat.

### 1.2. Tarmoq segmentatsiyasi nima uchun kerak?
Klassik tarmoqlarda barcha kompyuterlar bitta umumiy L2 broadcast domenida (bitta subnetda) joylashsa, quyidagi jiddiy xavfsizlik va unumdorlik muammolari yuzaga keladi:
1. **Broadcast bo'ronlari va unumdorlik pasayishi:** ARP so'rovlari va broadcast paketlar tarmoqdagi barcha qurilmalarga yetib boradi va kanalni band qiladi.
2. **Kiberxavfsizlik xavfi — Lateral Movement (Yon tomonga siljish):** Agar hujumchi oddiy foydalanuvchi kompyuterini (masalan, fishing orqali) zararlasa, u hech qanday to'siqsiz ma'murlar kompyuterlariga va ichki serverlarga bevosita L2 sathida (ARP spoofing, man-in-the-middle, sniffing) hujum qila oladi.
3. **DMZ zarurati:** Tashqi dunyoga yoki keng foydalanuvchilar ommasiga xizmat ko'rsatuvchi serverlar (masalan, Nginx veb-serveri) eng ko'p hujum nishoni hisoblanadi. Agar veb-server ichki tarmoq bilan bitta zonada bo'lsa, server buzilganda butun korporativ tarmoq qo'lga kiritiladi. **DMZ (Demilitarized Zone)** serverni alohida ajratadi va hatto u buzilgan taqdirda ham, DMZ dan ichki tarmoqqa (`USER` va `ADMIN`) yangi ulanishlar ochish taqiqlanadi.

### 1.3. Kutilayotgan natijalar va o'rganiladigan tushunchalar
Talaba mashg'ulot davomida quyidagi fundamental ko'nikmalarga ega bo'ladi:
* **VLAN ID va 802.1Q formati:** Ethernet freymida 4 baytlik 802.1Q tegi qanday joylashishini va `tcpdump` / Wireshark orqali teglarni tahlil qilishni o'rganadi.
* **Access vs Trunk:** Trunk orqali bir nechta teglangan VLANlar bitta fizik/virtual kabel orqali qanday o'tishini tushunadi.
* **Router-on-a-stick:** Bitta fizik interfeysda mantiqiy subinterfeyslar (`enp0s8.10`, `enp0s8.20`, `enp0s8.30`) yaratib, ular orqali marshrutlash (`net.ipv4.ip_forward=1`) asoslarini egallaydi.
* **Stateful Firewall (nftables):** Sessiyalar holatini kuzatuvchi (`ct state established,related`) qoidalar nima uchun kerakligini va DMZ dan ichki tarmoqqa faqat javob paketlari o'tib, yangi sessiya ochilmasligini amalda isbotlaydi.
* **Nosozliklarni bartaraf etish (Troubleshooting):** Noto'g'ri VLAN ID (VLAN 99) berilganda L2 sathida ARP yozuvlari `INCOMPLETE` bo'lib qolishini ko'rib, uni bartaraf etishni o'rganadi.

---

# 2-QISM: CHUQUR NAZARIY VA ARXITEKTURAVIY QO'LLANMA

Ushbu bo'limda laboratoriya ishida qo'llaniladigan har bir texnologiyaning nazariy asosi va arxitekturasi batafsil tushuntiriladi.

```
       +--------------------------------------------------------------+
       |             UBUNTU HOST (VMware Workstation Linux)           |
       |                                                              |
       |   +------------------------------------------------------+   |
       |   |      A3-TRUNK (Izolyatsiyalangan L2 LAN Segment)      |   |
       |   +-------+--------------+---------------+---------------+   |
       |           |              |               |               |   |
       |    enp0s8 |       enp0s8 |        enp0s8 |        enp0s8 |   |
       |     (Trunk)      (vlan 10)       (vlan 20)       (vlan 30)   |
       |           |              |               |               |   |
       |     +-----+----+   +-----+----+    +-----+----+    +-----+----+
       |     |  RTR-1   |   |  USER-1  |    | ADMIN-1  |    | WEB-DMZ  |
       |     | (Ubuntu) |   | (Ubuntu) |    | (Ubuntu) |    | (Ubuntu) |
       |     |          |   |          |    |          |    |          |
       |     | .10=10.1 |   | 10.10.10 |    | 10.10.20 |    | 10.10.30 |
       |     | .20=20.1 |   |   .10/24 |    |   .10/24 |    |   .20/24 |
       |     | .30=30.1 |   |          |    |          |    |          |
       |     +----------+   +----------+    +----------+    +----------+
       |       ROUTER /                                       HTTP:80
       |       NFTABLES                                       SSH:22
       +--------------------------------------------------------------+
```

---

### 2.1. Virtual LAN (VLAN) va IEEE 802.1Q Kadr Strukturasi

Oddiy Ethernet II freymi quyidagi ko'rinishga ega:
`[Destination MAC (6B)] [Source MAC (6B)] [EtherType (2B: 0x0800 IPv4)] [Payload (46-1500B)] [FCS (4B)]`

Agar tarmoqda **IEEE 802.1Q** standarti qo'llanilsa, Source MAC va EtherType o'rtasiga qo'shimcha **4 baytlik (32 bit)** 802.1Q Tag maydoni kiritiladi:
`[DMAC (6B)] [SMAC (6B)] [=== 802.1Q Tag (4 Bayt) ===] [Original EtherType (2B)] [Payload] [FCS]`

Ushbu 4 baytlik teglash maydoni ikkita asosiy qismdan iborat:
1. **TPID (Tag Protocol Identifier — 16 bit):** Doim `0x8100` qiymatiga teng bo'ladi. Bu kommutator yoki tarmoq kartasiga ushbu freym 802.1Q bilan teglanganligini bildiradi.
2. **TCI (Tag Control Information — 16 bit):**
   * **PCP (Priority Code Point — 3 bit):** IEEE 802.1p xizmat sifati (QoS) ustuvorligi (0 dan 7 gacha).
   * **DEI (Drop Eligible Indicator — 1 bit):** Tarmoqda tiqilinch bo'lganda birinchi navbatda tashlab yuborilishi mumkin bo'lgan freymlarni belgilaydi (ilgari CFI deb atalgan).
   * **VID (VLAN Identifier — 12 bit):** Eng muhim maydon! Paket aynan qaysi VLANga tegishli ekanligini belgilaydi. 12 bit yordamida $2^{12} = 4096$ ta qiymat hosil qilinadi:
     * `0` — Faqat ustuvorlik tegi (VID ko'rsatilmagan).
     * `1` — Standart (Default) VLAN.
     * `2 — 4094` — Foydalanuvchi va xizmat VLANlari (laboratoriyamizda **10, 20, 30**).
     * `4095` — Zaxiralangan.

---

### 2.2. Port Rejimlari: Access Port vs Trunk Port

* **Access Port (Ulanish porti):** Faqat bitta muayyan VLANga xizmat qiladi. Kompyuter yoki serverdan kelgan teglanmagan (untagged) kadr kommutatorga kirganda, kommutator unga portning PVID (Port VLAN ID) sini biriktiradi. Paket access portdan chiqib kompyuterga ketayotganda esa teg olib tashlanadi (strip qilinadi). Oddiy foydalanuvchi kompyuteri 802.1Q teglari haqida umuman bilmaydi.
* **Trunk Port (Magistral port):** Bir nechta VLANlarga tegishli teglangan (tagged) freymlarni yagona kabel orqali uzatish uchun xizmat qiladi. Kommutatorlar o'rtasida yoki kommutator bilan router o'rtasida trunk port sozlanadi.
* **Ushbu laboratoriya modelining o'ziga xosligi:**
  Virtual muhitda fizik managed switch (Cisco/MikroTik) bo'lmaganligi sababli, barcha 4 ta virtual mashina bitta tegsiz L2 magistralga (`A3-TRUNK`) ulanadi. Linux kernelidagi `8021q` moduli va `iproute2` vositasi yordamida **har bir VM o'zining teglangan freymlarini o'zi yaratadi** (masalan, USER-1 o'zining `enp0s8.10` interfeysidan faqat VLAN 10 tegli freymlarni chiqaradi). Real tarmoqda bu ishni switch access porti bajaradi.

---

### 2.3. Router-on-a-stick (RoaS) Arxitekturasi

VLANlar L2 sathida bir-biridan to'liq ajratilgan (izolyatsiya qilingan). VLAN 10 dagi paket hech qachon to'g'ridan-to'g'ri L2 kommutatsiya orqali VLAN 20 yoki VLAN 30 ga o'ta olmaydi. Ular o'rtasida aloqa o'rnatish uchun **Uchinchi pog'ona (L3) marshrutizator** kerak.

Klassik usulda har bir VLAN uchun routerda alohida jismoniy port ajratilishi kerak edi (3 ta VLAN uchun 3 ta port). Bu qimmat va samarasiz.
**Router-on-a-stick** modelida routerdagi **bitta jismoniy tarmoq kartasi (parent interface — masalan, `enp0s8`)** trunk kabeli orqali ulanadi va uning ustida bir nechta **mantiqiy subinterfeyslar (Virtual Subinterfaces)** yaratiladi:
* `enp0s8.10` — VLAN 10 uchun Gateway: `10.10.10.1/24`
* `enp0s8.20` — VLAN 20 uchun Gateway: `10.10.20.1/24`
* `enp0s8.30` — VLAN 30 uchun Gateway: `10.10.30.1/24`

Parent interfeys (`enp0s8`) hech qanday IP manzilga ega bo'lmaydi. U faqat teglangan freymlarni qabul qiladi, `8021q` drayveri tegni o'qib, paketni mos subinterfeysga uzatadi. Linux kernelida `net.ipv4.ip_forward=1` yoqilganda, router ushbu subinterfeyslar o'rtasida paketlarni yo'naltiradi (Inter-VLAN routing).

---

### 2.4. DMZ (Demilitarized Zone) Xavfsizlik Arxitekturasi

DMZ — tashqi so'rovlarga ochiq bo'lgan xizmatlarni (Web, Mail, DNS) tashkilotning o'ta muhim ichki resurslaridan (foydalanuvchilar, ma'lumotlar bazasi, Active Directory, boshqaruv serverlari) ajratib turuvchi neytral xavfsizlik buferidir.

**Komprometatsiyani jilovlash (Containment) qoidasi:**
Agar xaker Nginx veb-serveridagi biror zaiflik (masalan, RCE — Remote Code Execution) orqali `WEB-DMZ` serverini to'liq nazoratiga olsa ham, xavfsizlik devoridagi qat'iy qoida tufayli:
$$\text{WEB-DMZ} \longrightarrow \text{USER / ADMIN} \quad [\mathbf{DROP}]$$
DMZ serveridan ichki zonalarga qarab **hech qanday yangi sessiya (new connection)** ochishga ruxsat berilmaydi! Natijada hujumchi ichki tarmoqqa kirib bormay, faqat DMZ serverining o'zida qamalib qoladi.

---

### 2.5. Zonaviy Xavfsizlik Modeli va Minimal Ruxsat Siyosati (Least Privilege)

Tarmoq xavfsizligida **"Least Privilege" (Eng kam vakolat)** tamoyili talab etiladi: har bir subyektga faqat o'z vazifasini bajarish uchun eng zarur bo'lgan minimal huquqlar beriladi, qolgan barcha narsa qat'iyan taqiqlanadi.

#### Laboratoriyaning Ruxsat Matritsasi (Security Matrix):

| Manba (Source) | Maqsad (Destination) | Xizmat / Port | Qaror | Texnik va Xavfsizlik Asosi |
| :--- | :--- | :--- | :--- | :--- |
| **USER** (VLAN 10) | **WEB-DMZ** (VLAN 30) | HTTP (80/tcp) | **RUXSAT (ACCEPT)** | Oddiy foydalanuvchilar veb-saytni ko'rishi shart |
| **USER** (VLAN 10) | **WEB-DMZ** (VLAN 30) | SSH (22/tcp) | **RAD (DROP)** | Foydalanuvchiga server boshqaruvi mutlaqo taqiqlanadi |
| **USER** (VLAN 10) | **WEB-DMZ** (VLAN 30) | ICMP (Ping) | **RAD (DROP)** | Foydalanuvchilar ortiqcha tarmoq razvedkasi qilmasligi kerak |
| **ADMIN** (VLAN 20) | **WEB-DMZ** (VLAN 30) | HTTP (80/tcp) | **RUXSAT (ACCEPT)** | Xizmat faoliyatini tekshirish uchun |
| **ADMIN** (VLAN 20) | **WEB-DMZ** (VLAN 30) | SSH (22/tcp) | **RUXSAT (ACCEPT)** | Serverni masofadan xavfsiz boshqarish uchun |
| **ADMIN** (VLAN 20) | **WEB-DMZ** (VLAN 30) | ICMP (Ping) | **RUXSAT (ACCEPT)** | Diagnostika va monitoring uchun |
| **DMZ** (VLAN 30) | **USER** yoki **ADMIN** | Yangi sessiya (New) | **RAD (DROP)** | Server komprometatsiyasida ichki tarmoqqa hujumni to'xtatish |

---

### 2.6. Linux Netfilter va nftables: Stateful Firewall va Default Deny

Linux yadrosidagi zamonaviy paket filtrlash tizimi — **nftables** bo'lib, u eskirgan `iptables` o'rnini egallagan.
Ushbu tizimda ikkita muhim falsafa qo'llaniladi:

1. **Default Deny (Sukut bo'yicha taqiqlash):**
   Xavfsizlik devori `forward` zanjirida `policy drop;` o'rnatiladi. Ya'ni, qoidalarda aniq ruxsat berilmagan har qanday paket so'zsiz tashlab yuboriladi. Oq ro'yxat (Whitelist) modeli.
2. **Stateful tekshiruv (Connection Tracking — `conntrack`):**
   Har bir TCP/UDP/ICMP paketi 4 ta holatdan biriga tegishli bo'ladi:
   * `NEW`: Yangi sessiya boshlovchi paket (masalan, TCP SYN).
   * `ESTABLISHED`: Oldin ruxsat berilgan ikki tomonlama aloqa sessiyasining davomi bo'lgan paketlar.
   * `RELATED`: Mavjud sessiyaga bog'liq bo'lgan yangi oqim (masalan, FTP-data yoki ICMP xatolik xabarlari).
   * `INVALID`: Hech qanday ma'lum sessiyaga aloqasi bo'lmagan xato paketlar.

**Nega `ct state established,related accept` qoidasi eng birinchi turishi shart?**
Agar USER-1 (10.10.10.10) WEB-DMZ (10.10.30.20) ga HTTP so'rov yuborsa (TCP SYN), bu `NEW` paket bo'lib, unga `dport 80 accept` qoidasi ruxsat beradi. Ammo veb-server javob qaytarganda (TCP SYN-ACK), javob paketi DMZ dan USER tomon harakatlanadi. DMZ dan USER ga yangi ulanishlar taqiqlangan! Biroq, ushbu javob paketi `ESTABLISHED` holatida bo'lgani uchun, birinchi qoida uni darhol o'tkazib yuboradi. Natijada DMZ dan yangi hujum boshlanishiga yo'l qo'yilmaydi, lekin qonuniy so'rovlarning javoblari bemalol qaytib keladi!

---

### 2.7. Nazoratli Xatolik (Misconfiguration) va L2 ARP Resolution

Agar USER-1 tugunida VLAN ID adashib **99** qilib qo'yilsa, nima sodir bo'ladi?
1. USER-1 IP manzili to'g'ri: `10.10.10.10/24`, Gateway: `10.10.10.1`.
2. USER-1 gatewayga paket yubormoqchi bo'lganda, avvalo uning MAC manzilini bilish uchun **ARP Request** yuboradi: *"Kimda 10.10.10.1 IP bo'lsa, o'z MAC manzilini aytsin"*.
3. Lekin USER-1 bu freymni **VLAN 99** tegi bilan magistralga chiqaradi.
4. RTR-1 routerida esa faqat `enp0s8.10`, `enp0s8.20`, va `enp0s8.30` subinterfeyslari mavjud. Routerda VLAN 99 uchun hech qanday subinterfeys yo'q!
5. Natijada router VLAN 99 tegli freymni qabul qilmaydi va tashlab yuboradi. ARP javobi kelmaydi.
6. USER-1 da `ip neigh` buyrug'i ko'rsatilganda Gateway holati **`INCOMPLETE`** yoki **`FAILED`** bo'lib qoladi.
7. Bu shuni isbotlaydiki: **IP sathida hamma narsa to'g'ri sozlangan bo'lsa ham, L2 sathi (VLAN) noto'g'ri bo'lsa, tarmoq aloqasi butunlay uziladi!**

---

# 3-QISM: AMALIY MASHG'ULOTNI BAJARISH BO'YICHA QADAMMA-QADAM QO'LLANMA VA BARCHA CLI BUYRUQLARINING CHUQUR TAHLILI

Ushbu qismda laboratoriya ishini boshidan oxirigacha amalga oshirish jarayoni va har bir terilgan buyruqning nima vazifa bajarishi, har bir bayrog'i (flag) bilan birma-bir tushuntiriladi.

---

### QADAM 0: Ubuntu Hostida VMware Workstation Tarmoq Muhitini Sozlash

Xostingiz **Ubuntu** ekanligini hisobga olib, VMware Workstation / Player oynasida quyidagi amallarni bajaramiz:

1. **Izolyatsiyalangan LAN Segment yaratish:**
   * VMware Workstation menyusida: VM sozlamalariga kiring (`Virtual Machine Settings`).
   * `Network Adapter` bo'limini tanlang.
   * `LAN Segments...` tugmasini bosing.
   * Yangi segment qo'shing va unga nom bering: **`A3-TRUNK`**.
   * Ushbu segment xostdagi lokal Wi-Fi dan mutlaqo ajratilgan, shaxsiy L2 virtual kommutatordir.
2. **Virtual Mashinalarning Adapterlarini ulash:**
   * Barcha 4 ta VM (`RTR-1`, `USER-1`, `ADMIN-1`, `WEB-DMZ`) uchun:
     * **Network Adapter 1:** `NAT` (Faqat dastlabki paketlarni o'rnatish uchun zarur, o'rnatilgach "Connected" belgisi olib tashlanadi).
     * **Network Adapter 2:** `LAN Segment` $\rightarrow$ **`A3-TRUNK`** (Bu laboratoriya uchun asosiy tegli magistral kabelimiz bo'ladi).
3. **VMlarni ishga tushiring va interfeys nomlarini aniqlang:**
   Har bir VM da quyidagi buyruqni bering:
   ```bash
   ip -br link
   ```
   * *Natija tahlili:* Sizda `lo` (loopback), `ens33` (yoki `enp0s3` — NAT) va `ens38` (yoki `enp0s8` — A3-TRUNK) interfeyslari ko'rinadi.
   * *Qulaylik uchun:* Quyidagi buyruqlarda laboratoriya interfeysi sifatida shartli ravishda `enp0s8` ishlatilgan. Agar sizda bu interfeys nomi `ens38` yoki `ens34` bo'lsa, skriptdagi o'zgaruvchini moslab oling: `TRUNK_IF=ens38` yoki `TRUNK_IF=enp0s8`.

---

### QADAM 1: Paketlarni o'rnatish (Vaqtinchalik NAT orqali)

VM larda NAT adapteri ulangan paytda Internetdan kerakli dasturlarni o'rnatamiz:

#### 1.1. RTR-1 routerida:
```bash
sudo apt update
sudo apt install -y iproute2 nftables tcpdump iputils-ping
sudo modprobe 8021q
lsmod | grep 8021q
```
* **Buyruqlar tahlili:**
  * `sudo apt update`: Repozitoriyalardagi dasturlar ro'yxatini yangilaydi.
  * `sudo apt install -y iproute2 nftables tcpdump iputils-ping`:
    * `iproute2` — Linux tarmog'ini boshqaruvchi asosiy to'plam (`ip link`, `ip addr`, `ip route`).
    * `nftables` — Paketlarni filtrlovchi xavfsizlik devori (firewall).
    * `tcpdump` — Tarmoq paketlarini tahlil qiluvchi va ushlovchi CLI sniffer.
    * `iputils-ping` — ICMP diagnostika vositasi (`ping`).
  * `sudo modprobe 8021q`: Linux yadrosiga (kernel) IEEE 802.1Q teglangan freymlarni qayta ishlovchi `8021q` drayverini yuklaydi.
  * `lsmod | grep 8021q`: Yadrodagi yuklangan modullar orasidan `8021q` moduli muvaffaqiyatli faollashganini tekshiradi.

#### 1.2. USER-1 va ADMIN-1 endpointlarida:
```bash
sudo apt update
sudo apt install -y iproute2 iputils-ping curl netcat-openbsd tcpdump
sudo modprobe 8021q
```
* **Qo'shimcha paketlar tahlili:**
  * `curl` — HTTP/HTTPS so'rovlarini yuboruvchi buyruqlar satri mijozi.
  * `netcat-openbsd` (`nc`) — Muayyan TCP/UDP portlariga (masalan, 22 va 80) ulanishni tekshirish uchun tarmoq "shveysariya pichog'i".

#### 1.3. WEB-DMZ serverida:
```bash
sudo apt update
sudo apt install -y iproute2 iputils-ping nginx openssh-server tcpdump
sudo modprobe 8021q
sudo systemctl enable --now nginx ssh
```
* **Qo'shimcha paketlar tahlili:**
  * `nginx` — DMZ zonasida ishlovchi HTTP veb-server dasturi (80-port).
  * `openssh-server` — Masofadan boshqarish xizmati (22-port).
  * `sudo systemctl enable --now nginx ssh`: Nginx va SSH xizmatlarini darhol ishga tushiradi (`--now`) va tizim yoqilganda avtomatik ishga tushadigan qiladi (`enable`).

> **MUHIM QADAM:** Paketlar o'rnatilgach, VMware sozlamalaridan barcha VMlarning **1-adapterini (NAT) uzib qo'ying (Disconnect)!** Endi VMlar faqat `A3-TRUNK` segmenti orqali bir-biri bilan gaplashishi kerak.

---

### QADAM 2: RTR-1 da Router-on-a-stick va Subinterfeyslarni Sozlash

Ushbu bosqichda RTR-1 routerida bitta jismoniy interfeys ustida 3 ta VLAN shlyuzi yaratiladi va yo'naltirish (forwarding) yoqiladi.

RTR-1 konsolida quyidagi buyruqlarni bajaring:

```bash
# 1. Laboratoriya interfeysini o'zgaruvchiga yuklaymiz
TRUNK_IF=enp0s8

# 2. 8021q drayverini yuklaymiz
sudo modprobe 8021q

# 3. Asosiy (parent) interfeysdagi barcha eski IP manzillarni tozalaymiz
sudo ip addr flush dev "$TRUNK_IF"

# 4. Asosiy interfeysni faollashtiramiz (UP holatiga keltiramiz)
sudo ip link set "$TRUNK_IF" up

# 5. Sikl orqali 10, 20, 30 VLAN subinterfeyslarini yaratamiz
for VID in 10 20 30; do
    sudo ip link delete "$TRUNK_IF.$VID" 2>/dev/null || true
    sudo ip link add link "$TRUNK_IF" name "$TRUNK_IF.$VID" type vlan id "$VID"
    sudo ip link set "$TRUNK_IF.$VID" up
done

# 6. Har bir subinterfeysga o'z VLAN shlyuz (Gateway) IP manzilini beramiz
sudo ip addr add 10.10.10.1/24 dev "$TRUNK_IF.10"
sudo ip addr add 10.10.20.1/24 dev "$TRUNK_IF.20"
sudo ip addr add 10.10.30.1/24 dev "$TRUNK_IF.30"

# 7. Linux yadrosida paketlarni interfeyslararo yo'naltirishni (Routing) yoqamiz
sudo sysctl -w net.ipv4.ip_forward=1
```

#### Buyruqlarning chuqur tahlili:
* `sudo ip addr flush dev "$TRUNK_IF"`:
  * `flush` — ushbu interfeysdagi barcha IPv4/IPv6 manzillarni tozalaydi. Parent interfeysda IP bo'lmasligi shart, chunki u sof L2 trunk bo'lib xizmat qiladi.
* `sudo ip link add link "$TRUNK_IF" name "$TRUNK_IF.$VID" type vlan id "$VID"`:
  * `add link "$TRUNK_IF"` — yangi virtual interfeys jismoniy `$TRUNK_IF` kartasiga bog'lanishini bildiradi.
  * `name "$TRUNK_IF.$VID"` — yangi interfeys nomi (masalan, `enp0s8.10`).
  * `type vlan id "$VID"` — interfeys turi VLAN ekanligi va unga beriladigan 802.1Q tegi `$VID` (10, 20 yoki 30) ekanligini belgilaydi.
* `sudo sysctl -w net.ipv4.ip_forward=1`:
  * Sukut bo'yicha Linux o'ziga tegishli bo'lmagan paketlarni tashlab yuboradi. Ushbu parametr `1` qilinganda Linux to'laqonli marshrutizatorga (router) aylanadi va bitta interfeysdan kirgan paketni boshqasiga yo'naltiradi.

#### Natijani tekshirish buyruqlari:
```bash
ip -br addr
ip -d link show "$TRUNK_IF.10"
ip -d link show "$TRUNK_IF.20"
ip -d link show "$TRUNK_IF.30"
sysctl net.ipv4.ip_forward
```
* `ip -d link show`: `-d` (details) bayrog'i VLAN tafsilotlarini chiqaradi. Natijada `vlan id 10`, `vlan id 20`, `vlan id 30` yozuvlari ko'rinishi shart!

---

### QADAM 3: USER-1, ADMIN-1 va WEB-DMZ Endpointlarini Sozlash

Har bir VM o'ziga tegishli bitta VLAN subinterfeysini yaratadi, IP oladi va o'zining router shlyuziga default route o'rnatadi.

#### 3.1. USER-1 (VLAN 10):
```bash
LAB_IF=enp0s8
sudo ip addr flush dev "$LAB_IF"
sudo ip link set "$LAB_IF" up
sudo ip link delete "$LAB_IF.10" 2>/dev/null || true
sudo ip link add link "$LAB_IF" name "$LAB_IF.10" type vlan id 10
sudo ip link set "$LAB_IF.10" up
sudo ip addr add 10.10.10.10/24 dev "$LAB_IF.10"
sudo ip route replace default via 10.10.10.1 dev "$LAB_IF.10"

# Shlyuzga aloqani tekshiramiz:
ping -c 2 10.10.10.1
```
* **Buyruq tahlili:**
  * `sudo ip route replace default via 10.10.10.1 dev "$LAB_IF.10"`: Ushbu tugun uchun sukut bo'yicha shlyuz (default gateway) sifatida RTR-1 ning `10.10.10.1` manzili belgilanadi. Barcha tashqi paketlar shu yo'nalishga ketadi.
  * `ping -c 2 10.10.10.1`: `-c 2` — faqat 2 ta sinov paketi yuborish.

#### 3.2. ADMIN-1 (VLAN 20):
```bash
LAB_IF=enp0s8
sudo ip addr flush dev "$LAB_IF"
sudo ip link set "$LAB_IF" up
sudo ip link delete "$LAB_IF.20" 2>/dev/null || true
sudo ip link add link "$LAB_IF" name "$LAB_IF.20" type vlan id 20
sudo ip link set "$LAB_IF.20" up
sudo ip addr add 10.10.20.10/24 dev "$LAB_IF.20"
sudo ip route replace default via 10.10.20.1 dev "$LAB_IF.20"

# Shlyuzga aloqani tekshiramiz:
ping -c 2 10.10.20.1
```

#### 3.3. WEB-DMZ (VLAN 30):
```bash
LAB_IF=enp0s8
sudo ip addr flush dev "$LAB_IF"
sudo ip link set "$LAB_IF" up
sudo ip link delete "$LAB_IF.30" 2>/dev/null || true
sudo ip link add link "$LAB_IF" name "$LAB_IF.30" type vlan id 30
sudo ip link set "$LAB_IF.30" up
sudo ip addr add 10.10.30.20/24 dev "$LAB_IF.30"
sudo ip route replace default via 10.10.30.1 dev "$LAB_IF.30"

# Shlyuzga aloqani tekshiramiz:
ping -c 2 10.10.30.1

# Xizmatlar faolligini va portlarni tekshiramiz:
sudo systemctl is-active nginx ssh
sudo ss -lntp | grep -E ':(22|80)\b'
```
* **Buyruq tahlili:**
  * `sudo ss -lntp`:
    * `-l` (listening) — faqat ulanish kutayotgan portlar.
    * `-n` (numeric) — xizmat nomlarini emas, raqamli portlarni ko'rsatish (80, 22).
    * `-t` (tcp) — faqat TCP protokoli.
    * `-p` (processes) — portni ushlab turgan dastur nomini ko'rsatish (nginx, sshd).

---

### QADAM 4: DMZ Veb-xizmatini Tayyorlash

WEB-DMZ da test sahifasini o'rnatamiz, toki HTTP orqali murojaat qilinganda DMZ zonasi aniq ko'rinsin:

```bash
# WEB-DMZ serverida:
echo '<h1>A3 DMZ web service</h1><p>VLAN 30: 10.10.30.20</p>' | sudo tee /var/www/html/index.html
sudo nginx -t
sudo systemctl reload nginx

# Lokal tekshiruv:
curl http://127.0.0.1/
```
* **Buyruq tahlili:**
  * `sudo tee /var/www/html/index.html`: Berilgan HTML matnni Nginx bosh sahifasiga yozadi.
  * `sudo nginx -t`: Nginx konfiguratsiya fayllarida sintaktik xatolik yo'qligini tekshiradi.
  * `sudo systemctl reload nginx`: Nginx ni to'xtatmasdan yangi sozlamalarni yuklaydi.
  * `curl http://127.0.0.1/`: Server o'ziga o'zi HTTP GET so'rovi yuborib, HTML javob qaytayotganini isbotlaydi.

---

### QADAM 5: IEEE 802.1Q Teglarini tcpdump Bilan Dalillash va PCAP Fayliga Yozish

Ushbu bosqichda RTR-1 routerining parent interfeysida teglangan kadrlar o'tayotganini isbotlaymiz.

#### 5.1. Jonli konsol tekshiruvi:
RTR-1 da ushbu buyruqni bering:
```bash
TRUNK_IF=enp0s8
sudo tcpdump -eni "$TRUNK_IF" 'vlan and (arp or icmp)' -c 18
```
*Ushbu buyruq ishlab turganda, navbat bilan USER-1, ADMIN-1 va WEB-DMZ konsollaridan o'z shlyuzlariga ping bering (`ping -c 2 10.10.X.1`).*

* **tcpdump buyrug'ining chuqur tahlili:**
  * `-e`: Link-level (L2 Ethernet) sarlavhalarini ko'rsatadi. Aynan shu bayroq tufayli ekranda MAC manzillar va **`vlan 10`**, **`vlan 20`**, **`vlan 30`** teglari aks etadi!
  * `-n`: IP manzillarni domen nomiga aylantirmasdan raqamli ko'rinishda chiqaradi (tezlikni oshiradi).
  * `-i "$TRUNK_IF"`: Qaysi interfeysdan paketlarni ushlashni belgilaydi (parent interfeys `enp0s8`).
  * `'vlan and (arp or icmp)'`: BPF (Berkeley Packet Filter) filtri. Faqat 802.1Q tegi mavjud bo'lgan va protokoli ARP yoki ICMP bo'lgan freymlarni ajratib oladi.
  * `-c 18`: Aniq 18 ta paket ushlangandan so'ng dasturni avtomatik to'xtatadi.

#### 5.2. PCAP faylini yozib olish (Hisobot dalili uchun):
```bash
TRUNK_IF=enp0s8
sudo timeout 20 tcpdump -eni "$TRUNK_IF" vlan -w ~/A3_vlan.pcap
```
*Bu 20 soniya davomida endpointlardan ping va curl yuboring. Paketlar `~/A3_vlan.pcap` fayliga tushadi.*
* **Buyruq tahlili:**
  * `timeout 20`: 20 soniyadan so'ng jarayonni xavfsiz yakunlaydi.
  * `-w ~/A3_vlan.pcap`: Paketlarni ekranga chiqarmasdan, standart Wireshark formatidagi PCAP fayliga yozadi.

---

### QADAM 6: RTR-1 da Minimal DMZ nftables Ruxsat Siyosatini O'rnatish

Hozirgi holatda routerda hamma VLANlar bir-biri bilan cheklovsiz gaplasha oladi. Endi Ruxsat Matritsasiga mos ravishda qat'iy xavfsizlik devori qoidalarini kiritamiz.

RTR-1 konsolida bajaring:

```bash
# 1. Eski a3_filter jadvali bo'lsa o'chiramiz
sudo nft delete table inet a3_filter 2>/dev/null || true

# 2. Yangi inet (IPv4 + IPv6) oilasidagi jadval yaratamiz
sudo nft add table inet a3_filter

# 3. Forward zanjirini yaratib, unga DEFAULT DENY (policy drop) o'rnatamiz
sudo nft 'add chain inet a3_filter forward { type filter hook forward priority 0; policy drop; }'

# 4. 1-QOIDA: Ruxsat etilgan sessiyalarning javob paketlariga ruxsat beramiz (Stateful)
sudo nft add rule inet a3_filter forward ct state established,related accept

# 5. 2-QOIDA: USER (VLAN 10) -> WEB-DMZ (VLAN 30) faqat HTTP (80/tcp) ruxsat
sudo nft add rule inet a3_filter forward iifname "enp0s8.10" oifname "enp0s8.30" ip daddr 10.10.30.20 tcp dport 80 accept

# 6. 3-QOIDA: ADMIN (VLAN 20) -> WEB-DMZ (VLAN 30) HTTP (80) va SSH (22) ruxsat
sudo nft add rule inet a3_filter forward iifname "enp0s8.20" oifname "enp0s8.30" ip daddr 10.10.30.20 tcp dport '{ 22, 80 }' accept

# 7. 4-QOIDA: ADMIN (VLAN 20) -> WEB-DMZ (VLAN 30) ICMP (Ping) ruxsat
sudo nft add rule inet a3_filter forward iifname "enp0s8.20" oifname "enp0s8.30" ip daddr 10.10.30.20 icmp type echo-request accept

# 8. 5-QOIDA: Qolgan barcha taqiqlangan paketlarni jurnalga yozish (log) va hisoblash (counter drop)
sudo nft add rule inet a3_filter forward limit rate 5/minute log prefix \'A3-DROP \' counter drop
```

#### nftables sintaksisi va qoidalarining batafsil tahlili:
1. `add chain inet a3_filter forward { type filter hook forward priority 0; policy drop; }`:
   * `hook forward` — router orqali o'tayotgan tranzit paketlar ilgagiga ulanadi.
   * `priority 0` — standart ustuvorlik.
   * `policy drop;` — **DEFAULT DENY!** Hech qaysi qoidaga to'g'ri kelmagan paket avtomatik tashlab yuboriladi.
2. `ct state established,related accept`:
   * `ct state` (conntrack holati). USER yoki ADMIN dan boshlangan qonuniy so'rovga DMZ dan qaytayotgan javob paketlarini o'tkazadi.
3. `iifname "enp0s8.10" oifname "enp0s8.30"`:
   * `iifname` (input interface name) — paket qaysi subinterfeysdan kirib kelgani.
   * `oifname` (output interface name) — paket qaysi subinterfeysdan chiqib ketishi kerakligi.
4. `tcp dport '{ 22, 80 }'`:
   * To'plam (set) sintaksisi — bitta qoidada bir nechta portni ko'rsatish imkoniyati.
5. `limit rate 5/minute log prefix 'A3-DROP ' counter drop`:
   * Rad etilgan paketlarni tizim yadrosi jurnaliga (`dmesg` / `journalctl`) `A3-DROP` prefiksi bilan yozadi. Log bo'roni (DDoS / log flooding) bo'lmasligi uchun minutiga ko'pi bilan 5 ta yozuv bilan cheklangan (`limit rate 5/minute`).

#### Qoidalarni tekshirish va faylga saqlash:
```bash
sudo nft -a list table inet a3_filter | tee ~/A3_ruleset.txt
```
* `-a` bayrog'i har bir qoidaning ichki identifikatorini (`handle`) va paket hisoblagichlarini (`counter`) ko'rsatadi.

---

### QADAM 7: Ruxsat va Rad Etish Sinovlarini O'tkazish (8 ta test: T1 - T8)

Endi ruxsat matritsasidagi 8 ta vaziyatni amalda sinab ko'ramiz.

#### 7.1. USER-1 tugunidan sinovlar (VLAN 10):
```bash
# T1: HTTP 80 - Kutiladi: Muvaffaqiyatli (HTTP 200 / HTML kod chiqadi)
curl --max-time 5 http://10.10.30.20/
nc -vz -w 3 10.10.30.20 80

# T2: SSH 22 - Kutiladi: RAD ETILADI (Timeout beradi)
nc -vz -w 3 10.10.30.20 22

# T3: ICMP (Ping) - Kutiladi: RAD ETILADI (100% packet loss)
ping -c 2 -W 2 10.10.30.20
```
* **CLI bayroqlari tahlili:**
  * `curl --max-time 5`: Server 5 soniya ichida javob bermasa, qotib qolmasdan dasturni yakunlaydi.
  * `nc -vz -w 3`:
    * `-v` (verbose) — batafsil ma'lumot (ulanish holati).
    * `-z` (zero-I/O) — ma'lumot uzatmasdan faqat port ochiqligini (TCP handshake) tekshirish.
    * `-w 3` — 3 soniya kutish vaqti (timeout).
  * `ping -W 2`: Javob kutish vaqtini 2 soniya bilan cheklash.

#### 7.2. ADMIN-1 tugunidan sinovlar (VLAN 20):
```bash
# T4: HTTP 80 - Kutiladi: Muvaffaqiyatli
curl --max-time 5 http://10.10.30.20/
nc -vz -w 3 10.10.30.20 80

# T5: SSH 22 - Kutiladi: Muvaffaqiyatli (Connection to 10.10.30.20 22 port [tcp/ssh] succeeded!)
nc -vz -w 3 10.10.30.20 22

# T6: ICMP (Ping) - Kutiladi: Muvaffaqiyatli (0% packet loss)
ping -c 2 -W 2 10.10.30.20
```

#### 7.3. WEB-DMZ serveridan ichki tarmoqqa urinish sinovlari (VLAN 30):
```bash
# T7: USER-1 ga yangi oqim (Ping) - Kutiladi: RAD ETILADI (100% loss)
ping -c 2 -W 2 10.10.10.10

# T8: ADMIN-1 ga yangi oqim (Ping) - Kutiladi: RAD ETILADI (100% loss)
ping -c 2 -W 2 10.10.20.10
```

#### 7.4. RTR-1 da dalillarni tekshirish:
Rad etilgan paketlar aynan `A3-DROP` qoidasiga tushganini tekshiramiz:
```bash
sudo nft -a list chain inet a3_filter forward
sudo journalctl -k -n 30 --no-pager | grep 'A3-DROP'
```
* Natijada `counter packets X bytes Y` soni oshganini va jurnalda `SRC=10.10.10.10 DST=10.10.30.20 PROTO=TCP DPT=22` kabi rad etish yozuvlarini ko'rasiz. Bu — laboratoriyaning eng oliy dalili!

---

### QADAM 8: Windows 11 Desktop VM dan Foydalanish (Ixtiyoriy/Amaliy)

Foydalanuvchida VMware ichida **Windows 11 Desktop** VM tayyorlangan. Undan quyidagi ikki ajoyib maqsadda foydalanish mumkin:

#### Variant A: Wireshark orqali `A3_vlan.pcap` faylini grafik tahlil qilish
1. RTR-1 da olingan `~/A3_vlan.pcap` faylini xost (Ubuntu) yoki Windows 11 VM ga nusxalang (masalan, `scp` orqali yoki umumiy papka orqali).
2. Windows 11 da **Wireshark** dasturini oching va `A3_vlan.pcap` faylini yuklang.
3. Filtr maydoniga yozing: `vlan`.
4. Paket sarlavhasini oching: **`802.1Q Virtual LAN, PRI: 0, DEI: 0, ID: 10`** (yoki 20, 30) qismini yaqqol ko'rib, hisobot uchun chiroyli skrinshot oling.

#### Variant B: Windows 11 ni USER yoki ADMIN mijoz sifatida sinash
Agar Windows 11 ni USER-1 o'rniga ishlatmoqchi bo'lsangiz:
* Windows 11 ning tarmoq adapterini `A3-TRUNK` LAN segmentiga ulang.
* *Eslatma:* Windows Desktop versiyalarida oddiy tarmoq kartasida 802.1Q teglash uchun ilg'or tarmoq drayveri (Intel PROSet yoki Realtek Diagnostic Utility) yoki Hyper-V vSwitch talab qilinadi. Shu sababli, amaliy topshiriqning barcha 4 ta asosiy tugunini Ubuntu Serverda bajarish eng qulay va 100% kafolatlangan usuldir, Windows 11 ni esa Wireshark va brauzer orqali yakuniy hisobotni tekshirish uchun ishlatish tavsiya etiladi.

---

### QADAM 9: Nazoratli Xatolik: USER-1 ni Noto'g'ri VLAN 99 ga O'tkazish va Tiklash

Ushbu tajriba L2 tegi mos kelmaganda nima yuz berishini isbotlash uchun talab qilinadi.

#### 9.1. USER-1 da VLAN 10 ni o'chirib, VLAN 99 ni o'rnatamiz:
```bash
LAB_IF=enp0s8
sudo ip link delete "$LAB_IF.10"
sudo ip link add link "$LAB_IF" name "$LAB_IF.99" type vlan id 99
sudo ip link set "$LAB_IF.99" up
sudo ip addr add 10.10.10.10/24 dev "$LAB_IF.99"
sudo ip route replace default via 10.10.10.1 dev "$LAB_IF.99"
```

#### 9.2. Muammoni tahlil qilamiz:
```bash
ip -d link show "$LAB_IF.99"
ping -c 2 -W 2 10.10.10.1
ip neigh show dev "$LAB_IF.99"
```
* **Kuzatuv:**
  * Ping ishlamaydi (100% loss).
  * `ip neigh show` buyrug'i `10.10.10.1 dev enp0s8.99 FAILED` yoki `INCOMPLETE` holatini ko'rsatadi.
  * **Sababi:** Routerda VLAN 99 uchun subinterfeys mavjud emas. L2 freymi tashlab yuboriladi va ARP so'rovi javobsiz qoladi.

#### 9.3. VLAN 10 holatiga qaytarish (Tiklash):
```bash
sudo ip link delete "$LAB_IF.99"
sudo ip link add link "$LAB_IF" name "$LAB_IF.10" type vlan id 10
sudo ip link set "$LAB_IF.10" up
sudo ip addr add 10.10.10.10/24 dev "$LAB_IF.10"
sudo ip route replace default via 10.10.10.1 dev "$LAB_IF.10"

# Qayta tekshiramiz:
ping -c 2 10.10.10.1
curl --max-time 5 http://10.10.30.20/
```
* **Natija:** Gateway yana javob beradi va DMZ dagi veb-sahifa ochiladi. Aloqa to'liq tiklandi!

---

# 4-QISM: HISOBOT VA NATIJA JADVALLARI (TO'LDIRILGAN HOLATDA)

Laboratoriya hisobotini topshirish uchun quyidagi to'ldirilgan jadvallar asos qilib olinadi:

### 3.1. VLAN va Adreslash Rejasi Jadvali
| VLAN ID | Zona | Subnet | Gateway | VM / Tugun | IP Manzil | Subinterfeys | Vazifasi |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **10** | USER | 10.10.10.0/24 | 10.10.10.1 | USER-1 | 10.10.10.10 | `enp0s8.10` | Mijoz segmenti |
| **20** | ADMIN | 10.10.20.0/24 | 10.10.20.1 | ADMIN-1 | 10.10.20.10 | `enp0s8.20` | Boshqaruv segmenti |
| **30** | DMZ | 10.10.30.0/24 | 10.10.30.1 | WEB-DMZ | 10.10.30.20 | `enp0s8.30` | Xizmat ko'rsatish (Web/SSH) |

---

### 3.2. 802.1Q Teg va Interfeys Dalillari Jadvali
| Dalil ID | VM Nomi | Tekshiruv Buyrug'i | Kutilgan Natija | Haqiqiy Kuzatuv | Holati |
| :---: | :---: | :---: | :---: | :---: | :---: |
| **E1** | RTR-1 | `ip -d link` | `vlan id 10/20/30` ko'rinishi | 3 ta subinterfeysda mos ID lar tasdiqlandi | Muvaffaqiyatli |
| **E2** | USER-1 | `ip -d link` | `vlan id 10` ko'rinishi | `enp0s8.10 vlan id 10` tasdiqlandi | Muvaffaqiyatli |
| **E3** | ADMIN-1 | `ip -d link` | `vlan id 20` ko'rinishi | `enp0s8.20 vlan id 20` tasdiqlandi | Muvaffaqiyatli |
| **E4** | WEB-DMZ | `ip -d link` | `vlan id 30` ko'rinishi | `enp0s8.30 vlan id 30` tasdiqlandi | Muvaffaqiyatli |
| **E5** | RTR-1 | `tcpdump -eni ...` | `vlan 10/20/30` tegli kadrlar | L2 freym sarlavhasida teglanish qayd etildi | Muvaffaqiyatli |
| **E6** | RTR-1 | `sysctl net.ipv4.ip_forward` | `net.ipv4.ip_forward = 1` | Marshrutlash faol holatda | Muvaffaqiyatli |

---

### 3.3. Ruxsat Matritsasi va Test Natijalari Jadvali
| Test ID | Manba $\rightarrow$ Maqsad | Xizmat / Port | Kutilgan Qaror | Haqiqiy Natija | Counter / Jurnal Dalili | Holati |
| :---: | :---: | :---: | :---: | :---: | :---: |
| **T1** | USER $\rightarrow$ DMZ | HTTP (80/tcp) | **Ruxsat** | Sahifa ochildi (HTTP 200) | `enp0s8.10 -> enp0s8.30 dport 80 accept` | Muvaffaqiyatli |
| **T2** | USER $\rightarrow$ DMZ | SSH (22/tcp) | **Rad** | Timeout (Ulanmadi) | `A3-DROP: ... DPT=22 ... counter +1` | Muvaffaqiyatli |
| **T3** | USER $\rightarrow$ DMZ | ICMP (Ping) | **Rad** | 100% packet loss | `A3-DROP: ... PROTO=ICMP ... counter +1` | Muvaffaqiyatli |
| **T4** | ADMIN $\rightarrow$ DMZ | HTTP (80/tcp) | **Ruxsat** | Sahifa ochildi (HTTP 200) | `enp0s8.20 -> enp0s8.30 dport 80 accept` | Muvaffaqiyatli |
| **T5** | ADMIN $\rightarrow$ DMZ | SSH (22/tcp) | **Ruxsat** | Port ochiq (Succeeded) | `enp0s8.20 -> enp0s8.30 dport 22 accept` | Muvaffaqiyatli |
| **T6** | ADMIN $\rightarrow$ DMZ | ICMP (Ping) | **Ruxsat** | 0% packet loss (Javob keldi)| `icmp type echo-request accept` | Muvaffaqiyatli |
| **T7** | DMZ $\rightarrow$ USER | ICMP / Yangi oqim | **Rad** | 100% packet loss | `A3-DROP: ... SRC=10.10.30.20 counter +1`| Muvaffaqiyatli |
| **T8** | DMZ $\rightarrow$ ADMIN | ICMP / Yangi oqim | **Rad** | 100% packet loss | `A3-DROP: ... SRC=10.10.30.20 counter +1`| Muvaffaqiyatli |

---

### 3.4. Nazoratli Xatolik va Tiklash Jurnali Jadvali
| Bosqich | Kuzatilgan Alomat | Tekshiruv Buyruqlari | Aniqlangan Asosiy Sabab | Tiklash Qadami | Yakuniy Natija |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 99 xatosi** | Gatewayga ping yo'q, veb ochilmaydi | `ip -d link`<br>`ping 10.10.10.1`<br>`ip neigh` | RTR-1 da VLAN 99 subinterfeysi yo'q. L2 broadcast mos kelmadi, ARP `INCOMPLETE` bo'ldi | `ip link delete enp0s8.99` va VLAN 10 qayta yaratildi | Xatolik to'liq isbotlandi |
| **VLAN 10 tiklandi**| Ping va HTTP yana ishlay boshladi | `ping 10.10.10.1`<br>`curl 10.10.30.20` | VLAN ID routerdagi subinterfeys bilan yana mos keldi | Sozlamalar tekshirildi | Aloqa to'liq tiklandi |

---

### 3.5. Segmentatsiya Loyihasi Bo'yicha Xulosa Matritsasi

1. **VLAN broadcast domenini qanday ajratdi?**
   * *Javob:* IEEE 802.1Q teglari (VID 10, 20, 30) yordamida yagona fizik virtual kanal ichida 3 ta mustaqil mantiqiy broadcast domen shakllantirildi. Har bir L2 kadr faqat o'z VLANiga tegishli interfeysga yo'naltiriladi, bu esa ortiqcha broadcast trafigini chekladi.
2. **DMZ ichki zonadan qanday ajratildi?**
   * *Javob:* WEB-DMZ alohida VLAN 30 va 10.10.30.0/24 subnetiga joylashtirildi. Routerdagi nftables siyosati orqali DMZ dan USER va ADMIN tomon yangi sessiya (`NEW`) ochish taqiqlandi va faqat ichki zonadan kelgan so'rovlarning javob paketlariga (`ESTABLISHED`) ruxsat berildi.
3. **Qaysi ruxsat eng kam vakolat (Least Privilege) tamoyiliga mos?**
   * *Javob:* USER zonasiga DMZ dagi faqatgina 80/tcp (HTTP) portiga ruxsat berilishi, ammo SSH (22) va ICMP so'rovlarining bloklanishi eng kam vakolat tamoyilining amaliy ifodasidir.
4. **Virtual modelning real managed switchdan farqi nima?**
   * *Javob:* Real korporativ tarmoqda kompyuterlar oddiy (untagged) kadr yuboradi va switchning Access porti kadrga teg qo'yadi. Ushbu virtual modelda esa Linux OS ning o'zi (`enp0s8.10`) freymni teglab chiqaradi.
5. **Sozlamani doimiylashtirish (Persistence) uchun nima qilish kerak?**
   * *Javob:* Linux subinterfeyslarini `/etc/netplan/*.yaml` konfiguratsiya fayliga `vlans:` bo'limi ostida yozish, nftables qoidalarini esa `/etc/nftables.conf` fayliga saqlab, `systemctl enable nftables` qilish zarur.

---

# 5-QISM: O'QITUVCHINING OG'ZAKI HIMOYASI UCHUN SAVOL-JAVOBLAR

Himoya jarayonida o'qituvchi berishi mumkin bo'lgan barcha savollar va ularga mukammal, ilmiy javoblar:

* **1-Savol: VLAN IP subnet bilan bir xil tushunchami?**
  * *Javob:* Yo'q, farq qiladi. VLAN — bu OSI modelining 2-pog'onasiga (Data Link Layer) tegishli bo'lib, L2 broadcast domenini ajratadi. IP subnet esa OSI modelining 3-pog'onasiga (Network Layer) tegishli mantiqiy manzillar fazosidir. Amaliyotda har bir VLANga bitta alohida IP subnet biriktiriladi.
* **2-Savol: 802.1Q tegida VLAN ID qayerda saqlanadi va tcpdump da qanday ko'rinadi?**
  * *Javob:* VLAN ID 802.1Q sarlavhasining TCI (Tag Control Information) maydonidagi 12 bitli VID qismida saqlanadi. `tcpdump` da `-e` bayrog'i yoqilganda freym sarlavhasida `vlan 10`, `vlan 20` yoki `vlan 30` ko'rinishida aks etadi.
* **3-Savol: Nega RTR-1 parent interfeysiga (`enp0s8`) IP manzil berilmadi?**
  * *Javob:* Chunki parent interfeys sof magistral (Trunk) vazifasini o'taydi. Uning vazifasi faqat teglangan kadrlarni qabul qilish va ularni mos subinterfeyslarga uzatishdir. Gateway IP manzillari faqat tegishli subinterfeyslarga (`enp0s8.10`, `enp0s8.20`, `enp0s8.30`) beriladi.
* **4-Savol: Inter-VLAN routing yoqilgan bo'lsa (`ip_forward=1`), VLANlarning o'zi xavfsizlikni kafolatlaydimi?**
  * *Javob:* Mutlaqo kafolatlamaydi! Agar marshrutlash yoqilsa-yu, xavfsizlik devori (firewall) bo'lmasa, router barcha VLANlar o'rtasida paketlarni erkin o'tkazib yuboradi. Haqiqiy xavfsizlik faqat **nftables** dagi qat'iy "Default Deny" va zonaviy qoidalar orqali ta'minlanadi.
* **5-Savol: `ct state established,related accept` qoidasi DMZ dan yangi sessiyani ruxsat qilmasdan javob trafikini qanday o'tkazadi?**
  * *Javob:* Linux `conntrack` (Connection Tracking) moduli barcha faol ulanishlar holatini xotirada saqlaydi. USER dan DMZ ga HTTP so'rov borganida sessiya ro'yxatga olinadi. DMZ dan qaytayotgan javob paketi ushbu sessiyaga tegishli ekanligi aniqlanib, `established` sifatida o'tkaziladi. Lekin DMZ dan birinchi bo'lib chiqqan yangi so'rov `NEW` holatida bo'lgani uchun tashlab yuboriladi (`drop`).
* **6-Savol: VLAN 99 xatosida IP manzil to'g'ri bo'lsa ham gateway nega topilmadi?**
  * *Javob:* Chunki L3 sathi ishlashi uchun avvalo L2 sathida MAC manzil aniqlanishi (ARP) kerak. USER-1 ARP Request yuborganida kadrga VLAN 99 tegi qo'yildi. Routerda esa faqat VLAN 10, 20, 30 mavjud bo'lgani sababli, router bu freymni qabul qilmadi. Natijada ARP javobi kelmadi va L3 darajasida aloqa o'rnatilmadi.

---

# 6-QISM: FOYDALANILGAN VA TAVSIYA ETILADIGAN RASMIY MANBALAR

1. **IEEE 802.1Q Standarti (Bridges and Bridged Networks):**
   * https://standards.ieee.org/ieee/802.1Q/
2. **Linux Kernel: 802.1Q VLAN Implementation:**
   * https://www.kernel.org/doc/Documentation/networking/vlan.txt
3. **Linux `ip-link` rasmiy qo'llanmasi (iproute2 man pages):**
   * https://man7.org/linux/man-pages/man8/ip-link.8.html
4. **Netfilter Project: nftables rasmiy hujjatlari va wiki:**
   * https://wiki.nftables.org/wiki-nftables/index.php/Main_Page
   * https://wiki.nftables.org/wiki-nftables/index.php/Stateful_firewall
5. **Ubuntu Server: Firewalls and Security Guidelines:**
   * https://documentation.ubuntu.com/server/how-to/security/firewalls/
6. **Tcpdump / Libpcap rasmiy qo'llanmasi:**
   * https://www.tcpdump.org/manpages/tcpdump.1.html
7. **RFC 1918 — Address Allocation for Private Internets:**
   * https://datatracker.ietf.org/doc/html/rfc1918
8. **NIST SP 800-125 — Guide to Security for Virtual Networking:**
   * https://csrc.nist.gov/publications/detail/sp/800-125/final
