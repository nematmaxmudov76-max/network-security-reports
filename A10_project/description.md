# A10: DLP va NAC Siyosatlarini Ishlab Chiqish va Amalda Qo‘llash
**Kengaytirilgan Qo‘llanma va Laboratoriya Ko‘rsatmasi (1-Variant)**

---

## Kirish va Muhit Haqida Muhim Eslatma

Ushbu qo‘llanma **Tarmoq xavfsizligi** fanining **A10 amaliy mashg‘uloti** (1-variant: *Qurilma kirishi va ma’lumot yuborishni boshqarish*) bo‘yicha to‘liq, mukammal va tizimli qo‘llanma hisoblanadi.

> [!IMPORTANT]
> **Foydalanuvchi muhiti xususiyatlari:**
> 1. **Tarmoq muhiti:** Universitet Wi-Fi tarmog‘i emas, **shaxsiy/lokal Wi-Fi** ishlatiladi. Laboratoriya ichki tarmoqlari to‘liq izolyatsiya qilingan virtual segmentlarda ishlaydi. Dastlabki paketlarni o‘rnatish uchun lokal Wi-Fi orqali VM-larga Internet (NAT) beriladi, so‘ngra segmentlar ajratiladi.
> 2. **Asosiy tizim (Host OS):** Windows emas, **Ubuntu Linux (Desktop)**!
> 3. **Virtualizatsiya:** Ubuntu Host ichida **VMware Workstation Pro / Player for Linux** o‘rnatilgan.
> 4. **Virtual mashinalar (Guest OS):** VMware ichida **3 ta Ubuntu Server 24.04 LTS** virtual mashinasi ishga tushiriladi:
>    - `CLIENT` — Talaba/Mijoz ishchi stansiyasi.
>    - `RTR-1` — Tarmoq shlyuzi/marshrutizatori (NAC filtrlash tizimi bilan).
>    - `SRV-1` — O‘quv bo‘limi serveri (Remediation va DLP xizmatlari bilan).
> 5. **Fayl almashinuvi:** Natijalar va dalillarni Host (Ubuntu) ga ko‘chirish uchun VMware Shared Folder (`/mnt/hgfs/...`) yoki oddiy `scp` / lokal tarmoq usullari qo‘llaniladi.

---

## 1-BO‘LIM. Mavzuning Maqsadi va Konseptual Mohiyati

### 1.1. Asosiy Maqsad
Axborot xavfsizligida tizimlarni himoya qilish uchun ko‘p qatlamli mudofaa (**Defense-in-Depth**) tamoyili qo‘llaniladi. Ushbu laboratoriyaning bosh maqsadi — tarmoqqa ulanishni nazorat qiluvchi **NAC (Network Access Control)** va ma’lumotlar sizib chiqishini to‘suvchi **DLP (Data Loss Prevention)** tizimlarining mustaqil va bir-birini to‘ldiruvchi siyosatlarini ishlab chiqish hamda ularni virtual tarmoqda sinashdir.

Amaliyot davomida talaba quyidagilarni o‘rganadi:
1. **Qurilma huquqini ma’lumot huquqidan ajratish:** Qurilmaning tarmoqda bo‘lish huquqi boshqa, u yuborayotgan kontentga ruxsat berish butunlay boshqa mezonlar asosida amalga oshiriladi.
2. **Shart — Qaror — Dalil bog‘lanishi:** Xavfsizlik siyosatidagi har bir qaror (ruxsat berish, karantin, bloklash) aniq mezonlarga va tekshiriluvchi dalillarga asoslanishi kerak.
3. **Karantin va ruxsat mexanizmi:** Tarmoqqa yangi ulangan yoki talabga javob bermaydigan qurilmani to‘liq cheklab, faqat xatoni tuzatish sahifasiga yo‘naltirish, tekshiruvdan so‘ng esa huquqlarini ochish va qayta bekor qilish.
4. **DLP audit va bloklash rejimlari:** DLP tizimlarida qoidalarni to‘g‘ridan-to‘g‘ri bloklashga qo‘yish biznes jarayonlarini to‘xtatib qo‘yishi mumkin. Shu sababli *Audit* (monitoring) va *Enforce* (qatiy bloklash) rejimlarining farqini amalda ko‘rish.
5. **Xatoliklar balansi:** Noto‘g‘ri ijobiy (*False Positive*) va noto‘g‘ri salbiy (*False Negative*) hodisalarni tahlil qilish.

---

### 1.2. NAC va DLP: Taqqoslash va O‘zaro Bog‘liqlik

| Xususiyat | NAC (Network Access Control) | DLP (Data Loss Prevention) |
| :--- | :--- | :--- |
| **Himoyalash sathi (OSI)** | Tarmoq va Transport sathlari (L3 / L4) | Dasturiy / Amaliy sath (L7 - Application) |
| **Qaror obyekti** | **Subyekt va Qurilma** (IP, MAC, OS, xavfsizlik holati - posture) | **Obyekt va Kontent** (Fayl tarkibi, matn, kalit so‘zlar, regex, shaxsiy ma'lumotlar) |
| **Asosiy vazifasi** | Begona, sertifikatsiz yoki himoyalanmagan qurilmalarni tarmoqqa kiritmaslik | Ruxsatli qurilmadan maxfiy ma’lumotlar tashqariga chiqib ketishini to‘xtatish |
| **Ushbu laboratoriyada** | `RTR-1` routerida `nftables` orqali IP/Port filtrlash | `SRV-1` serverida Python HTTP xizmati orqali so‘rov tanasini tekshirish |
| **Tipik qarorlar** | Karantin (faqat 8080 port), Ruxsat (8000 va 8080 port), Drop | ALLOW (200), AUDIT (200), BLOCK (403), REJECT (400/413/415) |

#### Nima uchun ikkalasi ham zarur?
- **Faqat NAC bo‘lsa:** Ruxsat berilgan talaba yoki xodim o‘z kompyuteridan o‘quv bo‘limining maxfiy hujjatlarini, shaxsiy identifikatorlarni bemalol yuklab yoki boshqa joyga sizdirishi mumkin. NAC kontent ichiga qaramaydi.
- **Faqat DLP bo‘lsa:** Tarmoqda xaker yoki begona qurilma bemalol yurishi, serverning boshqa zaifliklariga hujum qilishi, tarmoq trafigini tinglashi mumkin. DLP tarmoqqa ulanishni nazorat qilmaydi.
- **Birgalikda (NAC + DLP):** NAC begona qurilmani karantinga tiqadi. Qurilma tekshirilib, tarmoqqa kiritilgach, DLP uning yuborayotgan ma'lumotlarini qat'iy nazorat qiladi.

---

### 1.3. Laboratoriya Modelining Chegarasi (Scope & Limitations)
Ushbu laboratoriya ta’lim maqsadida soddalashtirilgan modelga asoslangan:
- **NAC modeli:** Bu yerda to‘liq korporativ 802.1X/RADIUS protokoli yoki avtomatik posture agenti (masalan, Cisco ISE yoki PacketFence agenti) emas, balki operator tomonidan qurilma dalillari (`posture.txt`) ko‘zdan kechirilib, `nftables` orqali IP manzil asosida ruxsat berish amalga oshiriladi.
- **DLP modeli:** Butun operatsion tizimdagi USB fleshkalar, elektron pochta yoki printerlarni emas, aynan bitta HTTP `/upload` xizmati orqali kelayotgan ochiq `text/plain` matnini tekshiradi.

Shu cheklangan model orqali talaba ikki tizimning tarmoqdagi o‘rnini va mantiqiy mustaqilligini yaqqol ko‘ra oladi.

---

## 2-BO‘LIM. Nazariy Asoslar va Siyosatlar Loyihasi

Xavfsizlik siyosati (*Security Policy*) — bu xavfsizlik mexanizmlari qanday ishlashini belgilovchi qonun-qoidalar to‘plamidir. Har bir siyosat quyidagi rasmiy komponentlarga ega bo‘lishi shart:
- **Aktiv (Asset):** Himoyalanayotgan resurs (masalan, talabalar ma’lumotlar bazasi, o‘quv rejalari).
- **Subyekt (Subject):** Resursga murojaat qilayotgan tomon (foydalanuvchi, qurilma, IP).
- **Qamrov (Scope):** Siyosat amal qiladigan aniq chegara (protokol, port, URL).
- **Shart (Condition):** Qoida qachon ishga tushishi (qurilma holati, kalit so‘z).
- **Amal (Action):** Shart bajarilganda qilinadigan ish (ruxsat, blok, karantin).
- **Istisno (Exception):** Qoidadan chetga chiqish tartibi va shartlari.
- **Mas’ul (Owner/Role):** Siyosatni tasdiqlovchi va bajaruvchi shaxslar.
- **Dalil (Evidence):** Qoida ishlaganini isbotlovchi log, xesh, hisoblagich.

---

### 2.1. DLP Siyosati: Loyiha `DLP-A10-v1`

- **Siyosat kodi:** `DLP-A10-v1`
- **Egasi:** O‘quv bo‘limi (Data Owner)
- **Bajaruvchi:** Tizim administratori (System Admin)
- **Hodisalarni nazorat qiluvchi:** Axborot xavfsizligi mutaxassisi (SOC/Security Officer)
- **Qamrov:** Faqat `POST http://10.100.20.20:8000/upload` manziliga keluvchi, hajmi 1 dan 65 536 baytgacha (64 KB) bo‘lgan, kodlanishi `UTF-8` va sarlavhasi `Content-Type: text/plain` bo‘lgan matnli so‘rovlar.

#### Qoidalar Ketma-ketligi va Mantiqiy Jadvali:
Dastlab so‘rovning texnik formati (D00) va siyosat konfiguratsiyasi (D04) tekshiriladi. Agar ular to‘g‘ri bo‘lsa, matn ichi tahlil qilinadi:

| Qoida ID | Shart (Trigger Condition) | Rejim: ENFORCE | Rejim: AUDIT | Tushuntirish |
| :---: | :--- | :--- | :--- | :--- |
| **D00** | Format, hajm yoki kodlash mos emas (hajm > 64KB, UTF-8 emas, sarlavha xato) | **400 / 413 / 415 REJECT** | **400 / 413 / 415 REJECT** | Texnik rad etish. Fayl saqlanmaydi. Bu maxfiylik emas, format xatosi. |
| **D04** | `policy.json` mavjud emas, buzilgan yoki noma’lum rejim | **503 REJECT (Policy Error)** | **503 REJECT (Policy Error)** | Tizim xavfsiz rejimda to‘xtaydi (Fail-Secure). Xatolik bo‘lsa ma’lumot qabul qilinmaydi. |
| **D01** | Matnda `[CONFIDENTIAL]` yozuvi mavjud (katta-kichik harf farqsiz) | **403 BLOCK** | **200 AUDIT** (saqlanadi) | Maxfiy belgi aniqlanganda enforce rejimida to‘xtatiladi, auditda faqat loglanadi. |
| **D02** | `TALABA_ID: STU-123456` ko‘rinishidagi kontekstli talaba identifikatori | **403 BLOCK** | **200 AUDIT** (saqlanadi) | Shaxsga doir ma'lumot regex orqali ushlanadi. |
| **D03** | D01 va D02 shartlariga tushmaydigan, qamrovga mos ochiq matn | **200 ALLOW** (saqlanadi) | **200 ALLOW** (saqlanadi) | Ochiq hujjat qabul qilinadi va `accepted/` papkasiga yoziladi. |

#### DLP Rejimlari: Audit vs Enforce
1. **Audit (Monitoring) rejimi:** DLP qoidalari ishlaydi, qoidaga tushgan fayllar aniqlanadi, xavfsizlik jurnaliga (log) ogohlantirish yoziladi, lekin ma’lumot bloklanmaydi (`HTTP 200 AUDIT`), qabul qilinadi.
   * *Maqsadi:* Yangi qoidani real muhitga kiritishdan oldin noto‘g‘ri ijobiy natijalarni aniqlash va biznes jarayonlarini buzmaslik.
2. **Enforce (Bloklash) rejimi:** Maxfiy ma’lumot aniqlanganda so‘rov qat’iy to‘xtatiladi (`HTTP 403 Forbidden BLOCK`), fayl serverga saqlanmaydi va xavfsizlik hodisasi qayd etiladi.

#### Xatoliklar Balansi (False Positive vs False Negative):
- **Noto‘g‘ri ijobiy (False Positive — Soxta xavf):** Aslida maxfiy bo‘lmagan, zararsiz hujjatning DLP tomonidan bloklanishi.
  * *Misol:* O‘qituvchi tayyorlagan qo‘llanmada talabalarga misol tariqasida `TALABA_ID: STU-123456` deb yozilgan bo‘lsa, D02 qoidasi buni haqiqiy talaba ma’lumoti deb o‘ylab bloklaydi.
  * *Oqibati:* Xodimlar yoki talabalar o‘z ishini bajarolmaydi, norozilik paydo bo‘ladi.
- **Noto‘g‘ri salbiy (False Negative — O‘tkazib yuborilgan xavf):** Aslida o‘ta maxfiy bo‘lgan ma’lumotning DLP e’tiboridan chetda qolib, tarmoqdan chiqib ketishi.
  * *Misol:* Faylda talabaning ismi, baholari va pasport raqami bor, lekin `[CONFIDENTIAL]` so‘zi yoki `TALABA_ID:` prefiksi qo‘yilmagan. D03 qoidasi bo‘yicha fayl ochiq deb topilib, ruxsat beriladi.
  * *Oqibati:* Maxfiy ma’lumotlar sizib chiqadi, qonunbuzarlik va moddiy/ma’naviy zarar kelib chiqadi.
- **Istisnolar boshqaruvi (Exceptions):** Talaba yoki xodim o‘zboshimchalik bilan blokni chetlab o‘tolmaydi. Istisno so‘rovi biznes zarurati, ma'lumot egasining rasmiy roziligi, aniq jo‘natuvchi/qabul qiluvchi IP, amal qilish muddati va fayl xeshi (SHA-256) ko‘rsatilgan holda beriladi.

---

### 2.2. NAC Siyosati: Loyiha `NAC-A10-v1`

- **Siyosat kodi:** `NAC-A10-v1`
- **Egasi:** Tarmoq xavfsizligi bo‘limi
- **Ijro mexanizmi:** `RTR-1` routeridagi `nftables` paket filtrlash tizimi.

#### NAC Holatlari va Tarmoq Huquqlari:
Qurilmaning tarmoqdagi hayotiy sikli 4 ta holatdan iborat:

```
[ N01: Karantin ] ---> ( Tekshiruv dalillari ) ---> [ N02: Posture Ko‘rik ]
        ^                                                    |
        | (Nomuvofiqlik / Muddat tugashi)                    v (Tasdiqlash)
[ N04: Bekor Qilish ] <---------------------------- [ N03: To‘liq Ruxsat ]
```

| Holat ID / Nomi | Tekshirish Sharti | Tarmoq Huquqlari (Forwarding) |
| :---: | :--- | :--- |
| **N01 — Karantin** | Yangi ulangan qurilma; `admitted` ro‘yxatida IP yo‘q; tasdiqlanmagan. | **Faqat `SRV-1:8080` (Remediation / Tiklash yo‘riqnomasi).** Boshqa barcha portlar (shu jumladan 8000) va tranzit trafik DROP qilinadi. |
| **N02 — Tekshiruv (Posture)** | Qurilma inventar talablariga javob beradi: Hostname mos, OS Ubuntu 24.04, UFW xavfsizlik devori faol. | Avtomatik ruxsat berilmaydi. Dalillar (`posture.txt`) operatorga taqdim etiladi. |
| **N03 — Ruxsat (Admitted)** | Operator dalillarni tasdiqladi; qurilma IP-si routerdagi `@admitted` to‘plamiga qo‘shildi. | **`SRV-1:8000` (Asosiy yuklash xizmati) va `8080` ochiq.** Tashqi Internet va boshqa tranzit tarmoqlar hanuz blokda. |
| **N04 — Bekor qilish (Revoke)** | Dars/ish vaqti tugadi, xavfsizlik buzilishi aniqlandi yoki qoida muddati o‘tdi. | IP manzil `@admitted` to‘plamidan chiqariladi. Qurilma zudlik bilan yana **N01 (Karantin)** holatiga qaytadi. |

#### nftables ning Ishlash Mexanizmi:
`RTR-1` routerida `nftables` orqali dinamik kirish boshqariladi:
- **`set admitted { type ipv4_addr; }`:** Ruxsat berilgan IP manzillar to‘plami. Boshida u bo‘sh bo‘ladi. Operator bitta buyruq bilan IP qo‘shishi yoki o‘chirishi mumkin.
- **Stateful Connection Tracking (`ct state`):** 
  - So‘rov jo‘natilganda: `ct state new,established counter accept`
  - Javob qaytayotganda: `ct state established counter accept`
  - Nega bu muhim? Ruxsat bekor qilinganda (`N04`), agar routerda shunchaki umumiy `ct state established accept` qoidasi tursa, mijoz avval ochilgan 8000-sessiyasini yopmasdan foydalanaverishi mumkin. `nac.nft` qoidalarida 8000-port uchun javob paketlarida ham `@admitted` a’zoligi tekshiriladi, natijada ruxsat olingan zahoti ochiq sessiya ham darhol to‘xtaydi!

---

## 3-BO‘LIM. Laboratoriya Topologiyasi va Muhitni Tayyorlash

### 3.1. Tarmoq Topologiyasi

```
+-------------------------------------------------------------------------+
|                  UBUNTU HOST (Asosiy Operatsion Tizim)                  |
|                        Lokal Wi-Fi (192.168.x.x)                        |
|                                                                         |
|  +-------------------------------------------------------------------+  |
|  |                VMware Workstation Pro / Player                    |  |
|  |                                                                   |  |
|  |   [ CLIENT VM ]              [ RTR-1 VM ]            [ SRV-1 VM ] |  |
|  |  Ubuntu Server 24.04    Ubuntu Server 24.04     Ubuntu Server 24.04| |
|  |        |                       |      |                       |   |  |
|  |   enp0s8/ens33            enp0s8/ens33 enp0s9/ens34      enp0s8/ens33| |
|  |  10.100.10.10/24         10.100.10.1/24 10.100.20.1/24  10.100.20.20| |
|  |        |                       |      |                       |   |  |
|  |        +------- LAN Segment ---+      +--- LAN Segment -------+   |  |
|  |                  "A10-CLIENT"               "A10-SERVER"          |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
```

> [!CAUTION]
> **O‘ta Muhim Qoida:**
> `CLIENT` va `SRV-1` virtual mashinalarini aslo bitta tarmoq segmentiga ulamang! Ular alohida izolyatsiyalangan segmentlarda bo‘lishi va oralaridagi barcha trafik faqat `RTR-1` routeri orqali o‘tishi shart. Aks holda paketlar routerni chetlab o‘tib (L2 orqali), NAC siyosati butunlay ishlamay qoladi!

---

### 3.2. IP Manzillar va Virtual Adapterlar Rejasi

| Virtual Mashina (VM) | VMware Virtual Segmenti | Interfeys Nomlanishi* | IP Manzil / Niqob | Standart Shlyuz (Gateway) |
| :--- | :--- | :--- | :--- | :--- |
| **CLIENT** | LAN Segment: `A10-CLIENT` | `enp0s8` (yoki `ens33`) | `10.100.10.10/24` | `10.100.20.0/24 via 10.100.10.1` |
| **RTR-1 (Router)** | LAN Segment: `A10-CLIENT` | `enp0s8` (yoki `ens33`) | `10.100.10.1/24` | — |
| **RTR-1 (Router)** | LAN Segment: `A10-SERVER` | `enp0s9` (yoki `ens34`) | `10.100.20.1/24` | `sysctl net.ipv4.ip_forward=1` |
| **SRV-1 (Server)** | LAN Segment: `A10-SERVER` | `enp0s8` (yoki `ens33`) | `10.100.20.20/24` | `10.100.10.0/24 via 10.100.20.1` |

*\*Eslatma:* Ubuntu Serverda VMware drayverlariga qarab interfeyslar `ens33`, `ens34` yoki `enp0s8`, `enp0s9` deb nomlanishi mumkin. Har bir VM-da `ip -br link` buyrug‘i orqali haqiqiy nomni aniqlab oling.

---

### 3.3. VMware Workstation-ni Ubuntu Hostda Sozlash

1. **Virtual tarmoq segmentlarini yaratish:**
   - VMware Workstation oynasida: `VM` -> `Settings` -> `Network Adapter` bo‘limiga kiring.
   - `LAN Segments` tugmasini bosing va 2 ta yangi segment qo‘shing:
     - `A10-CLIENT`
     - `A10-SERVER`
2. **Adapterlarni ulash:**
   - **CLIENT VM:** 1 ta tarmoq adapteri -> `LAN Segment: A10-CLIENT`
   - **SRV-1 VM:** 1 ta tarmoq adapteri -> `LAN Segment: A10-SERVER`
   - **RTR-1 VM:** 2 ta tarmoq adapteri:
     - 1-adapter -> `LAN Segment: A10-CLIENT`
     - 2-adapter -> `LAN Segment: A10-SERVER`
3. **Dastlabki paketlarni o‘rnatish davri (Lokal Wi-Fi orqali):**
   - Dastlab barcha 3 ta VM-da vaqtincha `NAT` adapteri yoqiladi (Ubuntu Hostning lokal Wi-Fi internetidan foydalanish uchun).
   - Paketlar o‘rnatilgach, NAT adapterlari o‘chiriladi yoki yuqoridagi LAN Segmentlariga o‘tkaziladi.

---

## 4-BO‘LIM. Bosqichma-bosqich Amaliy Qo‘llanma va CLI Buyruqlarining To‘liq Tahlili

Quyidagi barcha bosqichlarda bajariladigan buyruqlar va ularning har bir parametri alohida tushuntirib berilgan.

---

### 0-BOSQICH: Kerakli Paketlarni O‘rnatish (Internet/NAT vaqtincha yoqilganda)

Har bir VM konsolida `sudo` huquqi bilan quyidagi buyruqlar bajariladi:

#### 1. Barcha VM-larda (CLIENT, RTR-1, SRV-1):
```bash
sudo apt update
sudo apt install -y python3 curl iproute2 iputils-ping
```
**Buyruqlar tahlili:**
- `sudo`: Superuser (root) nomidan ma’muriy huquqlar bilan buyruqni bajarish.
- `apt update`: Ubuntu paketlar ombori (repository) indekslarini yangilash, paketlarning so‘nggi versiyalari haqida ma’lumot olish.
- `apt install -y`: Paketlarni o‘rnatish. `-y` (`--yes`) flagi barcha tasdiqlash so‘rovlariga avtomatik "Ha" (yes) deb javob berishni bildiradi.
- `python3`: Python dasturlash tili interpretatori (SRV-1 da DLP va Remediation serverlarini ishga tushirish uchun zarur).
- `curl`: HTTP/HTTPS so‘rovlarini terminaldan yuboruvchi dastur (CLIENT da test so‘rovlarini yuborish uchun).
- `iproute2`: Tarmoq interfeyslari, IP manzillar va marshrutlashni boshqaruvchi utilitalar to‘plami (`ip` buyrug‘i).
- `iputils-ping`: Tarmoq ulanishini ICMP orqali tekshiruvchi utilita (`ping` buyrug‘i).

#### 2. Faqat RTR-1 routerida:
```bash
sudo apt install -y nftables
```
**Buyruq tahlili:**
- `nftables`: Linux yadrosidagi zamonaviy paket filtrlash tizimi (iptables o‘rnini bosuvchi yangi avlod xavfsizlik devori). Aynan shu paket NAC siyosatini bajaradi.

#### 3. Faqat CLIENT VM-da:
```bash
sudo apt install -y ufw
```
**Buyruq tahlili:**
- `ufw` (*Uncomplicated Firewall*): Ubuntu-ning soddalashtirilgan xavfsizlik devori. CLIENT qurilmasining xavfsizlik holatini (posture) ta’minlash va tekshirish uchun o‘rnatiladi.

> [!NOTE]
> Paketlar o‘rnatilgach, VMware sozlamalaridan NAT adapterlarini uzing va VM-larni faqat o‘zlariga tegishli **LAN Segment**lariga ulang!

---

### 1-BOSQICH: Tarmoq Interfeyslari, IP Manzillar va Marshrutlashni Sozlash

Interfeys nomlarini aniqlash uchun barcha VM-larda:
```bash
ip -br link
```
**Buyruq tahlili:**
- `ip`: Tarmoq boshqaruvi utilitasi.
- `-br` (`-brief`): Natijani qisqa, ixcham va jadval ko‘rinishida chiqarish.
- `link`: 2-sath (L2) tarmoq interfeyslari va MAC manzillarini ko‘rsatish.

---

#### A. CLIENT VM Konsolida:
Root rejimiga o‘tamiz va tarmoqni sozlaymiz:
```bash
sudo -i
export TZ=UTC
IF=enp0s8
ip link set "$IF" up
ip addr replace 10.100.10.10/24 dev "$IF"
ip route replace 10.100.20.0/24 via 10.100.10.1 dev "$IF"
mkdir -p /root/A10
```

**Buyruqlar va parametrlar tahlili:**
- `sudo -i`: Root foydalanuvchisi sifatida to‘liq interaktiv login sessiyasini ochish (barcha keyingi buyruqlar root sifatida bajariladi).
- `export TZ=UTC`: Tizim vaqt zonasini xalqaro UTC ga o‘rnatish (barcha loglar va hisobotlar yagona vaqtda bo‘lishi uchun).
- `IF=enp0s8`: Shell o‘zgaruvchisi. Interfeys nomini belgilaydi (agar sizda `ens33` bo‘lsa, `IF=ens33` qiling).
- `ip link set "$IF" up`: Tarmoq interfeysini faollashtirish (yoqish - link UP holatiga keltirish).
- `ip addr replace 10.100.10.10/24 dev "$IF"`: Interfeysga `10.100.10.10` IP manzilini `/24` (255.255.255.0) niqobi bilan biriktirish. `replace` parametri agar avval IP bo‘lsa, xatolik bermasdan uning o‘rniga yangisini qo‘yadi.
- `ip route replace 10.100.20.0/24 via 10.100.10.1 dev "$IF"`: Serverlar tarmog‘iga (`10.100.20.0/24`) boruvchi marshrutni router orqali (`via 10.100.10.1`) yo‘naltirish.
- `mkdir -p /root/A10`: Laboratoriya ishchi katalogini yaratish. `-p` ota papkalar mavjud bo‘lmasa ham xatosiz yaratadi.

---

#### B. RTR-1 (Router) Konsolida:
```bash
sudo -i
export TZ=UTC
LAN_IF=enp0s8
SRV_IF=enp0s9
ip link set "$LAN_IF" up
ip link set "$SRV_IF" up
ip addr replace 10.100.10.1/24 dev "$LAN_IF"
ip addr replace 10.100.20.1/24 dev "$SRV_IF"
sysctl -w net.ipv4.ip_forward=1
mkdir -p /root/A10
nft list ruleset > /root/A10/rules-before.txt
```

**Buyruqlar va parametrlar tahlili:**
- `LAN_IF=enp0s8` va `SRV_IF=enp0s9`: Mijoz tomonga qaragan (1-segment) va Server tomonga qaragan (2-segment) interfeyslar o‘zgaruvchisi.
- `ip addr replace ...`: Routerning har ikkala interfeysiga tegishli shlyuz IP manzillarini tayinlash (`10.100.10.1` va `10.100.20.1`).
- `sysctl -w net.ipv4.ip_forward=1`: Linux yadrosida paketlarni bir tarmoq interfeysidan ikkinchisiga yo‘naltirish (IPv4 Routing/Forwarding) funksiyasini yoqish. Bu parametrsiz Linux router vazifasini bajara olmaydi va paketlarni tashlab yuboradi!
- `nft list ruleset > /root/A10/rules-before.txt`: NAC o‘rnatilishidan oldingi xavfsizlik devori qoidalarini faylga saqlab olish (taqqoslash dalili uchun).

---

#### C. SRV-1 (Server) Konsolida:
```bash
sudo -i
export TZ=UTC
IF=enp0s8
ip link set "$IF" up
ip addr replace 10.100.20.20/24 dev "$IF"
ip route replace 10.100.10.0/24 via 10.100.20.1 dev "$IF"
mkdir -p /root/A10/{out,accepted,remediation}
chmod 700 /root/A10
```

**Buyruqlar va parametrlar tahlili:**
- `ip addr replace 10.100.20.20/24 dev "$IF"`: Serverga `10.100.20.20` IP manzilini biriktirish.
- `ip route replace 10.100.10.0/24 via 10.100.20.1 dev "$IF"`: Mijozlar tarmog‘iga (`10.100.10.0/24`) javob qaytarish uchun yo‘lni router shlyuzi (`via 10.100.20.1`) orqali belgilash.
- `mkdir -p /root/A10/{out,accepted,remediation}`: Bash kengaytmasi orqali birdaniga 3 ta ichki papkani yaratish:
  - `out/` — DLP loglari va xizmat jurnallari uchun;
  - `accepted/` — DLP tomonidan ruxsat berilgan fayllar saqlanadigan joy;
  - `remediation/` — Karantindagi qurilmalar uchun veb-sahifa joyi.
- `chmod 700 /root/A10`: Katalog huquqlarini faqat root o‘qiy oladigan, yozadigan va kiradigan qilib cheklash (`rwx------`).

---

#### D. Boshlang‘ich Aloqani Tekshirish:
Hali `nftables` (NAC) qoidalari yoqilmagan paytda CLIENT konsolidan serverga ping yuborib ko‘riladi:
```bash
ping -c 3 10.100.20.20
```
**Buyruq tahlili:**
- `-c 3`: Faqat 3 ta ICMP Echo Request paketi yuborilsin va to‘xtasin.
- **Kutiladigan natija:** Ping muvaffaqiyatli o‘tadi (0% packet loss). Bu router marshrutlashni to‘g‘ri bajarayotganini isbotlaydi.

---

### 2-BOSQICH: CLIENT Qurilmasi Holatini (Posture) Yig‘ish

Haqiqiy NAC tizimlarida kompyuter tarmoqqa to‘liq ulanishidan oldin uning operatsion tizimi, xavfsizlik devori va yangilanishlari tekshiriladi.

CLIENT konsolida bajaramiz:
```bash
hostnamectl set-hostname A10-CLIENT-01
ufw default deny incoming
ufw default allow outgoing
ufw --force enable
{ date -u; hostnamectl; cat /etc/os-release; ip -br link; ufw status verbose; } > /root/A10/posture.txt
cat /root/A10/posture.txt
```

**Buyruqlar va parametrlar tahlili:**
- `hostnamectl set-hostname A10-CLIENT-01`: Qurilmaning tarmoqdagi nomini (hostname) siyosat talabiga binoan `A10-CLIENT-01` ga o‘zgartirish.
- `ufw default deny incoming`: Xavfsizlik devoriga tashqaridan keluvchi barcha ulanishlarni standart bo‘yicha taqiqlash (xavfsizlik talabi).
- `ufw default allow outgoing`: Kompyuterdan tashqariga chiquvchi trafikka ruxsat berish (HTTP test so‘rovlari chiqishi uchun).
- `ufw --force enable`: UFW xavfsizlik devorini yoqish (`--force` foydalanuvchidan qo‘shimcha tasdiq so‘ramasdan darhol yoqadi).
- `{ ... } > /root/A10/posture.txt`: Bir nechta buyruqlar natijasini bitta blok qilib, `posture.txt` dalil fayliga yozish:
  - `date -u`: UTC bo‘yicha aniq vaqt.
  - `hostnamectl`: Hostname va tizim arxitekturasi.
  - `cat /etc/os-release`: Operatsion tizim versiyasi (Ubuntu 24.04).
  - `ip -br link`: Qurilmaning MAC manzili.
  - `ufw status verbose`: Xavfsizlik devorining faol holati va qoidalari.
- `cat /root/A10/posture.txt`: Yaratilgan dalil fayli tarkibini ekranga chiqarish.

---

### 3-BOSQICH: RTR-1 da NAC Siyosatini (nftables) Ishga Tushirish

RTR-1 routerida kirish nazoratini yoqamiz.

#### 1. Qoidalar faylini yaratish:
```bash
cat > /root/A10/nac.nft <<'NFT'
table inet a10_nac {
    set admitted { 
        type ipv4_addr; 
    }
    chain forward {
        type filter hook forward priority 0; policy drop;

        # 1-Qoida: Karantindagi CLIENT uchun 8080 (Remediation) so'rovi
        iifname "enp0s8" oifname "enp0s9" ip saddr 10.100.10.10 ip daddr 10.100.20.20 tcp dport 8080 ct state new,established counter accept

        # 2-Qoida: 8080 portdan qaytuvchi javob trafigi
        iifname "enp0s9" oifname "enp0s8" ip saddr 10.100.20.20 ip daddr 10.100.10.10 tcp sport 8080 ct state established counter accept

        # 3-Qoida: Faqat admitted ro'yxatidagi IP uchun 8000 (DLP xizmati) so'rovi
        iifname "enp0s8" oifname "enp0s9" ip saddr @admitted ip daddr 10.100.20.20 tcp dport 8000 ct state new,established counter accept

        # 4-Qoida: 8000 portdan qaytuvchi javob trafigi (faqat admitted uchun!)
        iifname "enp0s9" oifname "enp0s8" ip saddr 10.100.20.20 ip daddr @admitted tcp sport 8000 ct state established counter accept

        # Standart bo'yicha qolgan hamma narsa DROP qilinadi
        counter drop
    }
}
NFT
```

**Qoidalar va sintaksis tahlili:**
- `table inet a10_nac`: IPv4 va IPv6 uchun yagona bo‘lgan `inet` oilasiga mansub `a10_nac` nomli jadval yaratish.
- `set admitted { type ipv4_addr; }`: Dinamik IPv4 to‘plami (ruxsat berilgan IP-lar shu yerga yoziladi).
- `chain forward`: Router orqali tranzit o‘tuvchi paketlar uchun zanjir.
- `hook forward priority 0; policy drop;`: Barcha o‘tuvchi paketlarni standart bo‘yicha tashlab yuborish (Drop — Fail-Safe yondashuv).
- `iifname` va `oifname`: Kiruvchi (*input interface*) va chiquvchi (*output interface*) tarmoq kartasi nomlari.
- `ip saddr` va `ip daddr`: Manba (*source*) va mo‘ljal (*destination*) IP manzillari.
- `tcp dport 8080` / `tcp sport 8080`: TCP mo‘ljal porti va manba porti.
- `ct state new,established`: Ulanishni kuzatish (*Connection Tracking*). Yangi ulanish (`new`) yoki davom etayotgan ulanish (`established`).
- `ip saddr @admitted`: Manba IP manzili `admitted` to‘plami ichida mavjud bo‘lishi shart!
- `counter`: Qoidaga to‘g‘ri kelgan paketlar va baytlar sonini sanab boruvchi hisoblagich (audit va tekshiruv uchun o‘ta muhim dalil).

#### 2. Sintaksisni tekshirish va qoidalarni yuklash:
```bash
nft -c -f /root/A10/nac.nft
nft -f /root/A10/nac.nft
nft -a list table inet a10_nac
```
**Buyruqlar tahlili:**
- `nft -c -f /root/A10/nac.nft`: `-c` (`--check`) flagi konfiguratsiyani faqat sintaktik xatolarga tekshiradi, lekin tizimga yuklamaydi. Xato bo‘lsa ko‘rsatadi.
- `nft -f /root/A10/nac.nft`: `-f` (`--file`) flagi qoidalarni o‘qib, to‘g‘ridan-to‘g‘ri Linux yadrosiga (kernel) yuklaydi.
- `nft -a list table inet a10_nac`: `-a` (`--handle`) flagi har bir qoidaning ichki unikal raqamini (handle) ko‘rsatgan holda jadvalni ekranga chiqaradi.

---

### 4-BOSQICH: SRV-1 da DLP va Remediation Xizmatlarini Yaratish va Ishga Tushirish

SRV-1 serverida test qabul qiluvchi DLP xizmati va Karantindagi mijozlar uchun yo‘riqnoma sahifasi yaratiladi.

#### 1. Python DLP Server kodini yaratish:
SRV-1 konsolida quyidagi buyruqlar orqali skript yaratiladi:
```bash
cd /root/A10
cat > dlp_server.py <<'PYCODE'
from http.server import HTTPServer, BaseHTTPRequestHandler
from pathlib import Path
import hashlib, json, re
from datetime import datetime, timezone

BASE = Path(__file__).resolve().parent

class Handler(BaseHTTPRequestHandler):
    def log_message(self, *args):
        # Odatiy konsol loglarini o'chirib, toza ushlaymiz
        pass

    def reply(self, status, decision, rule, body=b''):
        digest = hashlib.sha256(body).hexdigest() if body else None
        event = {
            'time': datetime.now(timezone.utc).isoformat(),
            'ip': self.client_address[0],
            'decision': decision,
            'rule': rule,
            'bytes': len(body),
            'sha256': digest
        }
        with (BASE / 'out/dlp.jsonl').open('a') as log:
            log.write(json.dumps(event) + '\n')
        data = json.dumps(event).encode()
        self.send_response(status)
        self.send_header('Content-Type', 'application/json')
        self.send_header('Content-Length', str(len(data)))
        self.end_headers()
        self.wfile.write(data)

    def do_POST(self):
        self.connection.settimeout(5)
        if self.path != '/upload':
            return self.reply(404, 'REJECT', 'PATH')
        if self.headers.get('Transfer-Encoding'):
            return self.reply(400, 'REJECT', 'D00_TRANSFER')
        try:
            size = int(self.headers.get('Content-Length', '0'))
        except ValueError:
            return self.reply(400, 'REJECT', 'D00_LENGTH')
        if not 1 <= size <= 65536:
            return self.reply(413, 'REJECT', 'D00_SIZE')
        if self.headers.get('Content-Type') != 'text/plain':
            return self.reply(415, 'REJECT', 'D00_TYPE')
        try:
            body = self.rfile.read(size)
            text = body.decode('utf-8')
        except (UnicodeDecodeError, OSError):
            return self.reply(415, 'REJECT', 'D00_ENCODING')
        if len(body) != size:
            return self.reply(400, 'REJECT', 'D00_TRUNCATED')

        # Siyosat rejimini o'qish (audit yoki enforce)
        try:
            mode = json.loads((BASE / 'policy.json').read_text())['mode']
            if mode not in ('audit', 'enforce'):
                raise ValueError('mode')
        except (OSError, ValueError, KeyError, TypeError):
            return self.reply(503, 'REJECT', 'D04_POLICY_ERROR')

        rule = 'D03'
        if '[CONFIDENTIAL]' in text.upper():
            rule = 'D01'
        elif re.search(r'\bTALABA_ID\s*:\s*STU-\d{6}\b', text, re.I):
            rule = 'D02'

        match = rule != 'D03'
        if match and mode == 'enforce':
            return self.reply(403, 'BLOCK', rule, body)

        name = hashlib.sha256(body).hexdigest() + '.txt'
        (BASE / 'accepted' / name).write_bytes(body)
        decision = 'AUDIT' if match else 'ALLOW'
        self.reply(200, decision, rule, body)

if __name__ == '__main__':
    HTTPServer(('10.100.20.20', 8000), Handler).serve_forever()
PYCODE
```

**Dastur kodi va DLP mantiqining tahlili:**
- `reply(...)`: Barcha hodisalarni `out/dlp.jsonl` fayliga JSON Lines formatida yozadi. Eng muhimi: xavfsizlik nuqtai nazaridan xom matn yoki talaba ID-si logga saqlanmaydi, faqat uning SHA-256 xeshi va metama’lumotlari yoziladi!
- `do_POST(...)`:
  - `self.path != '/upload'`: Faqat `/upload` manziliga qaraydi (qamrov nazorati).
  - `Content-Length`: 1 dan 65536 baytgacha bo‘lishi shart (D00 qoidasi).
  - `Content-Type`: Faqat `text/plain` qabul qilinadi.
  - `body.decode('utf-8')`: Agar UTF-8 bo‘lmasa, `415 Unsupported Media Type` beradi.
  - `policy.json`: Har bir so‘rovda qayta o‘qiladi. Bu serverni o‘chirib-yoqmasdan rejimni `audit` yoki `enforce` ga o‘tkazish imkonini beradi.
  - `[CONFIDENTIAL] in text.upper()`: D01 qoidasi — katta yoki kichik harflarda yozilgan maxfiy yozuvni ushlaydi.
  - `re.search(r'\bTALABA_ID\s*:\s*STU-\d{6}\b', text, re.I)`: D02 qoidasi — kontekstli talaba identifikatorini qidiradi.
  - `name = hashlib.sha256(body).hexdigest() + '.txt'`: Qabul qilingan ochiq fayllar o‘z xesh qiymati bilan `accepted/` papkasiga saqlanadi (yaxlitlik va hisobdorlik ta’minlanadi).

#### 2. Konfiguratsiyani tayyorlash va Xizmatlarni Fonda Ishga Tushirish:
```bash
cd /root/A10
printf '{"mode":"enforce"}\n' > policy.json
python3 -m py_compile dlp_server.py
python3 dlp_server.py > out/service.log 2>&1 &
echo $! > /run/a10-dlp.pid

printf 'A10: inventar, OS va UFW holatini operatorga korsating.\n' > remediation/index.html
python3 -m http.server 8080 --bind 10.100.20.20 --directory /root/A10/remediation > out/remediation.log 2>&1 &
echo $! > /run/a10-remediation.pid

ss -lntp '( sport = :8000 or sport = :8080 )'
```

**Buyruqlar va parametrlar tahlili:**
- `printf '{"mode":"enforce"}\n' > policy.json`: Siyosat rejimini dastlab `enforce` (qatiy bloklash) qilib belgilash.
- `python3 -m py_compile dlp_server.py`: Skriptda sintaktik xatolar (masalan, indentatsiya/bo‘sh joy xatolari) yo‘qligini tekshirish.
- `python3 dlp_server.py > out/service.log 2>&1 &`:
  - `> out/service.log`: Standart chiqishni (stdout) log faylga yo‘naltirish.
  - `2>&1`: Xatoliklar chiqishini (stderr) ham stdout bilan birga bitta faylga birlashtirish.
  - `&`: Buyruqni fonda (*background process*) ishga tushirish (terminal bo‘sh qoladi).
- `echo $! > /run/a10-dlp.pid`: `$!` — so‘nggi fonda ishga tushgan jarayonning unikal identifikatori (Process ID - PID). Uni keyinchalik xizmatni boshqarish/to‘xtatish uchun `.pid` faylga yozib qo‘yish.
- `python3 -m http.server 8080 --bind 10.100.20.20 --directory /root/A10/remediation ...`: Python-ning tayyor veb-server moduli yordamida `8080` portda, faqat `10.100.20.20` manzilida `remediation/` papkasi kontentini tarqatuvchi xizmatni fonda yoqish.
- `ss -lntp '( sport = :8000 or sport = :8080 )'`:
  - `ss`: Socket Statistics utilitasi (netstat o‘rniga).
  - `-l`: Faqat tinglayotgan (*listening*) portlarni ko‘rsatish.
  - `-n`: Port va IP nomlarini (DNS) raqamli ko‘rinishda chiqarish.
  - `-t`: Faqat TCP soketlari.
  - `-p`: Qaysi dastur (process) portni egallab turganini (PID va nomini) ko‘rsatish.
  - **Kutiladigan natija:** Ikkala port (`8000` va `8080`) ham `10.100.20.20` manzilida tinglayotgani ko‘rinishi kerak.

---

### 5-BOSQICH: CLIENT da Sinov Fayllarini Tayyorlash

CLIENT konsolida 6 ta har xil holatni sinovchi test fayllarini yaratamiz:
```bash
cd /root/A10
printf 'Ochiq elon: amaliy dars 10:00 da.\n' > public.txt
printf '[CONFIDENTIAL]\nSuniy ichki reja.\n' > secret.txt
printf 'TALABA_ID: STU-123456\nFaqat suniy test.\n' > student.txt
printf 'Format misoli: STU-123456; haqiqiy talaba yoq.\n' > tutorial.txt
cp secret.txt renamed_public.csv
printf '\377\376' > binary.dat
sha256sum public.txt secret.txt student.txt tutorial.txt renamed_public.csv binary.dat
```

**Fayllarning vazifasi va tahlili:**
1. `public.txt`: Hech qanday maxfiy belgi yoki identifikator yo‘q ochiq matn (D03 ruxsat berishi kerak).
2. `secret.txt`: Tarkibida `[CONFIDENTIAL]` maxfiy yorlig‘i bor (D01 bloklashi kerak).
3. `student.txt`: Tarkibida `TALABA_ID: STU-123456` kontekstli identifikatori bor (D02 bloklashi kerak).
4. `tutorial.txt`: Tarkibida shunchaki `STU-123456` yozilgan, lekin `TALABA_ID:` konteksti yo‘q. Bu qo‘llanma yoki tushuntirish matni (D03 ochiq deb o‘tkazishi kerak — Noto‘g‘ri ijobiylikni oldini olish testi).
5. `renamed_public.csv`: `secret.txt` faylining nusxasi, lekin nomi o‘zgartirilgan. Bu DLP fayl kengaytmasiga emas, ichidagi haqiqiy tarkibga qarayotganini isbotlash uchun.
6. `binary.dat`: Matn emas, ikkilik (binary UTF-16 BOM) baytlar. Bu D00 qoidasi bo‘yicha kodlash xatosini (`415 Unsupported Media Type`) keltirib chiqarishi kerak.
- `sha256sum ...`: Har bir faylning kriptografik xeshini hisoblash (kelgusida hisobotda solishtirish uchun).

---

### 6-BOSQICH: Amaliy Sinov 1 — Karantindagi Qurilma (N01)

Hozir CLIENT hali `admitted` ro‘yxatiga kiritilmagan. Routerning NAC qoidasi uni karantinda ushlab turibdi.

CLIENT konsolida bajaramiz:
```bash
cd /root/A10
curl --noproxy '*' -sS --max-time 3 http://10.100.20.20:8080/
curl --noproxy '*' -sS --max-time 3 -w '\nHTTP=%{http_code}\n' \
     -H 'Content-Type: text/plain' --data-binary @public.txt \
     http://10.100.20.20:8000/upload
```

**Buyruqlar va parametrlar tahlili:**
- `curl`: URL manzilga so‘rov yuborish.
- `--noproxy '*'`: Tizimdagi hech qanday proksi server ishlatilmasin, so‘rov to‘g‘ridan-to‘g‘ri tarmoqqa chiqsin.
- `-sS` (`--silent --show-error`): Yuklanish progress-barini yashirish (`-s`), lekin agar xatolik yuz bersa xabarni ko‘rsatish (`-S`).
- `--max-time 3`: Serverdan javob kutishning maksimal vaqti — 3 soniya (agar javob kelmasa kutib qotib qolmasdan to‘xtaydi).
- `-w '\nHTTP=%{http_code}\n'`: So‘rov yakunida server qaytargan HTTP javob kodini aniq formatda chiqarish.
- `-H 'Content-Type: text/plain'`: HTTP so‘rov sarlavhasiga ma’lumot turi matn ekanini yozish.
- `--data-binary @public.txt`: Faylni hech qanday o‘zgartirishsiz, asl bayt holatida POST so‘rovi tanasida yuborish.

**Natijalar tahlili:**
1. **8080-port so‘rovi:** Tiklash yo‘riqnomasi matni muvaffaqiyatli qaytadi:  
   `"A10: inventar, OS va UFW holatini operatorga korsating."`
2. **8000-port so‘rovi:** Timeout bo‘ladi (odatda `curl: (28) Failed to connect to ... Connection timed out`) va `HTTP=000` chiqadi!
   - *Izoh:* `HTTP=000` bu server qaytargan status kodi emas! Bu `curl` ning server bilan umuman bog‘lana olmaganini bildiradi. Router paketi tashlab yuborgani (DROP) sababli so‘rov serverga yetib ham bormadi!

RTR-1 konsolida buni tekshiramiz:
```bash
nft -a list chain inet a10_nac forward
```
- **Dalil:** Routerning oxiridagi `counter drop` hisoblagichi oshganini ko‘rasiz!

---

### 7-BOSQICH: Amaliy Sinov 2 — Operator Ko‘rigi va Ruxsat Berish (N02 -> N03)

1. Operator (talaba) CLIENT dagi `posture.txt` faylini ko‘zdan kechiradi:
   - Hostname: `A10-CLIENT-01` (to‘g‘ri);
   - OS: `Ubuntu 24.04 LTS` (to‘g‘ri);
   - UFW: `Status: active` (to‘g‘ri);
   - IP: `10.100.10.10` (kutilgan manzil).
2. Shartlar to‘liq bajarilgani sababli, operator RTR-1 routerida ushbu IP-ga ruxsat beradi:
```bash
nft add element inet a10_nac admitted '{ 10.100.10.10 }'
{ date -u; nft list set inet a10_nac admitted; } > /root/A10/admission.txt
cat /root/A10/admission.txt
```
**Buyruqlar tahlili:**
- `nft add element inet a10_nac admitted '{ 10.100.10.10 }'`: `admitted` dinamik to‘plamiga CLIENT ning IP manzilini qo‘shish. Shu paytdan boshlab 3- va 4-qoidalar ishga tushadi va 8000-port ochiladi.
- `{ date -u; nft list set ...; } > ...`: Ruxsat berilgan vaqt va admitted to‘plami holatini dalil sifatida `admission.txt` ga saqlash.

---

### 8-BOSQICH: Amaliy Sinov 3 — Ruxsatli Qurilmadan DLP Sinovlari

Endi qurilma tarmoqqa kirdi (NAC ruxsat berdi). Navbat DLP tizimining kontentni qanday tekshirishini sinashga keldi!

CLIENT konsolida 6 ta faylni avtomatlashtirilgan tsiklda yuboramiz:
```bash
cd /root/A10
for file in public.txt secret.txt student.txt tutorial.txt renamed_public.csv binary.dat; do
    echo "TEST=$file"
    curl --noproxy '*' -sS --max-time 5 -w '\nHTTP=%{http_code}\n' \
         -H 'Content-Type: text/plain' --data-binary "@$file" \
         http://10.100.20.20:8000/upload
    echo
done 2>&1 | tee dlp-tests.txt
```

**Buyruq tahlili:**
- `for file in ...; do ... done`: Ro‘yxatdagi har bir fayl nomini olib, ketma-ket `curl` so‘rovini bajaruvchi Bash tsikli.
- `2>&1 | tee dlp-tests.txt`: Ham standart chiqishni, ham xatoliklarni ekranga chiqarish bilan bir vaqtda `dlp-tests.txt` fayliga yozib borish (`tee` buyrug‘i).

#### Kutilgan va Haqiqiy Natijalar Tahlili Jadvali:

| Fayl Nomi | HTTP Kod | DLP Qarori | Qoida ID | Sababi va Xavfsizlik Mantiqi |
| :--- | :---: | :---: | :---: | :--- |
| **`public.txt`** | **200** | **ALLOW** | **D03** | Maxfiy belgi yo‘q, ochiq e’lon. Server uni qabul qildi va `accepted/` ga saqladi. |
| **`secret.txt`** | **403** | **BLOCK** | **D01** | Tarkibida `[CONFIDENTIAL]` aniqlandi. Enforce rejimida qat’iy bloklandi. |
| **`student.txt`** | **403** | **BLOCK** | **D02** | `TALABA_ID: STU-123456` kontekstli identifikatori aniqlandi. Bloklandi. |
| **`tutorial.txt`** | **200** | **ALLOW** | **D03** | Matnda `STU-123456` bor, lekin `TALABA_ID:` konteksti yo‘q. Bu format namunasi bo‘lgani uchun DLP ruxsat berdi (Noto‘g‘ri ijobiylik oldi olindi). |
| **`renamed_public.csv`**| **403** | **BLOCK** | **D01** | Fayl nomi `.csv` qilib almashtirilgan bo‘lsa ham, DLP tarkibdagi `[CONFIDENTIAL]` yozuvini ko‘rib blokladi. (DLP fayl nomiga emas, kontentga qaraydi!). |
| **`binary.dat`** | **415** | **REJECT** | **D00_ENCODING** | Fayl matn emas, ikkilik baytlar (UTF-8 emas). Qamrovga to‘g‘ri kelmagani uchun texnik rad etildi. |

---

### 9-BOSQICH: Amaliy Sinov 4 — Audit va Enforce Rejimlarini Taqqoslash

DLP siyosatida rejim o‘zgarishini tekshiramiz.

#### 1. SRV-1 da Audit rejimiga o‘tish:
```bash
printf '{"mode":"audit"}\n' > /root/A10/policy.json
```
*Izoh:* `dlp_server.py` faylni har bir so‘rovda qayta o‘qiydi, shuning uchun serverni qayta ishga tushirish talab qilinmaydi!

#### 2. CLIENT da maxfiy faylni qayta yuborish:
```bash
curl --noproxy '*' -sS --max-time 5 -w '\nHTTP=%{http_code}\n' \
     -H 'Content-Type: text/plain' --data-binary @secret.txt \
     http://10.100.20.20:8000/upload
```
- **Kutiladigan natija:** `HTTP=200`, JSON javobida `{"decision": "AUDIT", "rule": "D01"}` keladi!
- *Tahlil:* Faylda maxfiy belgi bo‘lsa ham bloklanmadi, serverga qabul qilindi, lekin audit jurnalida xavf sifatida qayd etildi.

#### 3. Darhol Enforce rejimini qaytarish va qayta sinash:
SRV-1 da:
```bash
printf '{"mode":"enforce"}\n' > /root/A10/policy.json
```
CLIENT da yana o‘sha so‘rovni bajaramiz:
```bash
curl --noproxy '*' -sS --max-time 5 -w '\nHTTP=%{http_code}\n' \
     -H 'Content-Type: text/plain' --data-binary @secret.txt \
     http://10.100.20.20:8000/upload
```
- **Kutiladigan natija:** `HTTP=403`, `{"decision": "BLOCK", "rule": "D01"}`.
- *Dalil:* Bir xil fayl audit rejimida 200, enforce rejimida 403 natija berdi. Bu siyosat rejimining to‘g‘ri ishlayotganini amalda isbotlaydi.

---

### 10-BOSQICH: Amaliy Sinov 5 — Ruxsatni Bekor Qilish (N04 - Revoke)

Dars/mashg‘ulot yakunlandi yoki qurilmadan shubhali harakat sezildi. Operator qurilma ruxsatini bekor qiladi.

RTR-1 routerida bajaramiz:
```bash
nft delete element inet a10_nac admitted '{ 10.100.10.10 }'
{ date -u; nft list set inet a10_nac admitted; nft -a list chain inet a10_nac forward; } > /root/A10/revoked.txt
cat /root/A10/revoked.txt
```
**Buyruq tahlili:**
- `nft delete element ...`: CLIENT IP manzilini `admitted` to‘plamidan chiqarib tashlash.
- Natijani `revoked.txt` ga saqlash.

#### CLIENT da qayta tekshirish:
```bash
curl --noproxy '*' -sS --max-time 3 http://10.100.20.20:8080/
curl --noproxy '*' -sS --max-time 3 -w '\nHTTP=%{http_code}\n' \
     -H 'Content-Type: text/plain' --data-binary @public.txt \
     http://10.100.20.20:8000/upload
```
- **Kutiladigan natija:**
  - 8080-port: Yana ochiq (tiklash sahifasi ko‘rinadi).
  - 8000-port: Yana `HTTP=000` / Timeout beradi!
  - SRV-1 dagi `dlp.jsonl` logiga qarasangiz, ushbu oxirgi so‘rov umuman yetib kelmaganini ko‘rasiz (chunki router uni tarmoq sathidayoq DROP qildi).

---

### 11-BOSQICH: Dalillarni Yig‘ish va Ubuntu Hostga Ko‘chirish

Laboratoriya hisoboti uchun barcha dalillarni tartibga keltiramiz.

#### 1. SRV-1 da loglar va xeshlar ro‘yxatini tayyorlash:
```bash
cd /root/A10
cat out/dlp.jsonl
ls -lh accepted
sha256sum accepted/* > out/accepted-hashes.txt
cat out/accepted-hashes.txt
```
**Buyruqlar tahlili:**
- `cat out/dlp.jsonl`: Barcha DLP qarorlari qayd etilgan JSONL jurnalini ko‘rish.
- `sha256sum accepted/* > out/accepted-hashes.txt`: Server qabul qilib olgan ochiq fayllarning xeshlarini saqlash.

#### 2. RTR-1 da yakuniy qoidalar holatini olish:
```bash
nft -a list table inet a10_nac > /root/A10/nac-final.txt
```

#### 3. Dalillar arxivini yaratish (SRV-1 da):
```bash
cd /root/A10
tar -czf A10_dalillar_SRV1.tar.gz dlp_server.py policy.json out accepted remediation
```
**Buyruq tahlili:**
- `tar`: Fayllarni bitta arxivga jamlash va arxivdan chiqarish dasturi (*Tape Archive*).
- `-c` (`--create`): Yangi arxiv yaratish.
- `-z` (`--gzip`): Arxivni `gzip` algoritmi bilan siqish (hajmni kichraytirish).
- `-f` (`--file`): Yaratiladigan arxiv faylining nomini belgilash (`A10_dalillar_SRV1.tar.gz`).

#### 4. Dalillarni Ubuntu Hostga Ko‘chirish Usullari:

##### 1-USUL: VMware Shared Folders orqali (Tavsiya etiladi)
Ubuntu Hostda biror papkani (masalan, `~/A10_share`) VMware sozlamalarida Shared Folder qilib belgilang.
So‘ng SRV-1 ichida:
```bash
mkdir -p /mnt/a10share
vmhgfs-fuse .host:/A10_share /mnt/a10share -o allow_other
cp /root/A10/A10_dalillar_SRV1.tar.gz /mnt/a10share/
```
**Buyruq tahlili:**
- `vmhgfs-fuse`: VMware Workstation-ning Linux uchun umumiy papkalar drayveri (FUSE asosida).
- `.host:/A10_share`: Host tizimdagi umumiy papka nomi.
- `-o allow_other`: Barcha foydalanuvchilarga o‘qish/yozish huquqini berish.

##### 2-USUL: Ubuntu Host orqali to‘g‘ridan-to‘g‘ri SCP orqali olish
Agar vaqtincha Host va VM orasida tarmoq bo‘lsa yoki Host konsolidan:
```bash
# Ubuntu Host terminalida:
scp root@<VM_IP>:/root/A10/A10_dalillar_SRV1.tar.gz ~/Downloads/
```

---

## 5-BO‘LIM. Nazorat Savollari va Og‘zaki Himoyaga Tayyorgarlik

Ushbu bo‘limda laboratoriya himoyasida o‘qituvchi berishi mumkin bo‘lgan 10 ta asosiy savolga mukammal, ilmiy va amaliy javoblar keltirilgan.

### 1-Savol: NAC va DLP qaysi obyekt bo‘yicha qaror chiqaradi?
**Javob:**
- **NAC** subyekt va uning qurilmasi (L3/L4 sathida: IP manzil, MAC manzil, operatsion tizim holati, xavfsizlik devori mavjudligi — posture) bo‘yicha qaror chiqaradi.
- **DLP** obyekt va uzatilayotgan ma’lumot kontenti (L7 dasturiy sathida: fayl ichidagi kalit so‘zlar, regex shablonlar, ma'lumot tuzilishi va formati) bo‘yicha qaror chiqaradi.

### 2-Savol: NAC ruxsat bergan bo‘lsa ham, DLP nima sababdan faylni bloklashi mumkin?
**Javob:**
Chunki NAC faqat qurilmaning tarmoqqa kirishiga kafolat beradi, lekin u yuborayotgan ma’lumotning ichki maxfiyligini tekshirmaydi. Qurilma qonuniy va to‘liq himoyalangan bo‘lsa ham, undan maxfiy xizmat hujjati yoki talabalarning shaxsiy ma’lumotlari yuborilayotgan bo‘lsa, amaliy sathdagi DLP siyosati (`D01` yoki `D02` qoidasi) bu ma’lumotni ushlaydi va bloklaydi (`HTTP 403 Forbidden`).

### 3-Savol: `HTTP=000`, `HTTP=403` va `HTTP=415` statuslari o‘rtasida qanday farq bor?
**Javob:**
- **`HTTP=000`:** Bu server qaytargan HTTP kodi emas. Bu `curl` ning tarmoq sathida server bilan umuman TCP ulanish o‘rnata olmaganini (Timeout yoki Drop) bildiradi. Qaror **NAC/Router** sathida qabul qilingan.
- **`HTTP=403 (Forbidden):`** TCP ulanish o‘rnatildi, HTTP so‘rov serverga yetib bordi, lekin **DLP tizimi** matn ichida maxfiy ma’lumot (`[CONFIDENTIAL]` yoki `TALABA_ID`) borligini aniqlab, qasddan so‘rovni blokladi.
- **`HTTP=415 (Unsupported Media Type):`** So‘rov serverga yetib bordi, lekin uning formati siyosat qamroviga mos kelmadi (masalan, `text/plain` o‘rniga ikkilik fayl yuborildi yoki UTF-8 kodlanishida emas). Bu maxfiylik tufayli emas, texnik parametrlar mos kelmagani uchun berilgan rad javobidir.

### 4-Savol: Audit rejimida maxfiy faylga amalda nima bo‘ladi?
**Javob:**
Audit rejimida maxfiy ma’lumot aniqlangan taqdirda ham fayl bloklanmaydi. U server tomonidan muvaffaqiyatli qabul qilinadi (`HTTP 200 AUDIT`), diskka saqlanadi (`accepted/` papkasiga yoziladi), lekin xavfsizlik jurnaliga (`dlp.jsonl`) bu so‘rov maxfiy ekanligi va qaysi qoidaga tushgani haqida ogohlantirish qayd etiladi. Bu rejim qoidalarni ishlab chiqarish muhitiga kiritishdan oldin noto‘g‘ri ijobiy natijalarni o‘rganish uchun kerak.

### 5-Savol: Nima uchun fayl nomini o‘zgartirish (`renamed_public.csv`) D01 qarorini o‘zgartirmadi?
**Javob:**
Chunki haqiqiy DLP tizimlari fayl nomi yoki uning kengaytmasiga (extension) qarab emas, balki so‘rov tanasidagi haqiqiy baytlar va matn tarkibiga qarab qaror chiqaradi (*Deep Content Inspection*). Fayl nomi har qancha niqoblanmasin, uning ichida `[CONFIDENTIAL]` satri saqlanib qolgani sababli DLP uni birdek aniqlaydi va bloklaydi.

### 6-Savol: D02 qoidasida kontekstdan foydalanish qaysi xatoni kamaytiradi va qaysisini oshirishi mumkin?
**Javob:**
- **Kamaytiradi:** Noto‘g‘ri ijobiy (*False Positive*) xatoliklarni keskin kamaytiradi. Masalan, o‘quv qo‘llanmasida shunchaki `STU-123456` kodi uchrasa, u bloklanmaydi, chunki oldida `TALABA_ID:` konteksti yo‘q.
- **Oshirishi mumkin:** Noto‘g‘ri salbiy (*False Negative*) xatoliklarni oshirishi mumkin. Agar foydalanuvchi bilmasdan yoki qasddan `ID: STU-123456` yoki shunchaki `Talaba kodi: STU-123456` deb yozsa, regex kontekstga mos kelmay qoladi va maxfiy ma’lumot tizimdan ochiq holda sizib chiqib ketadi.

### 7-Savol: IP manzilga asoslangan `admitted` ro‘yxatini nima uchun to‘laqonli NAC deb bo‘lmaydi?
**Javob:**
Chunki IP manzil qurilmaning shaxsini va uning ishonchliligini to‘liq isbotlay olmaydi. Hujumchi tarmoqdagi ruxsat berilgan IP manzilni o‘z kompyuteriga statik qilib qo‘yib olishi (*IP Spoofing*) yoki tarmoq paketlarini soxtalashtirishi mumkin. Haqiqiy to‘laqonli NAC tizimlarida 802.1X protokoli, raqamli sertifikatlar (EAP-TLS), kompyuter apparat ta’minotining kriptografik identifikatorlari (TPM) va dinamik VLAN kommutatsiyasi qo‘llaniladi.

### 8-Savol: Ruxsatni bekor qilganda routerdagi umumiy `established` qoidasining xavfi nimada?
**Javob:**
Agar routerda shunchaki umumiy `ct state established accept` qoidasi bo‘lsa, qurilma ro‘yxatdan chiqarilgan taqdirda ham, u avval ochib olgan uzoq muddatli TCP ulanishlari (masalan, SSH sessiyasi yoki davom etayotgan fayl uzatish) yopilmay qoladi va ulanish uzilguncha trafik o‘taveradi. Shu sababli bizning `nac.nft` qoidalarimizda javob trafigida ham `@admitted` to‘plamiga a’zolik majburiy qilib tekshirilgan.

### 9-Savol: DLP istisnosini kim va qancha muddatga tasdiqlashi kerak?
**Javob:**
DLP istisnosini oddiy talaba yoki tizim administratori o‘zboshimchalik bilan tasdiqlay olmaydi. Istisno faqat **Ma’lumot Egasi (Data Owner)** yoki **Axborot xavfsizligi bo‘yicha mas’ul shaxs (CISO/Security Officer)** tomonidan tasdiqlanishi kerak. U doimiy bo‘lmasdan, qat’iy belgilangan qisqa muddatga (masalan, 2 soat yoki 1 ish kuni), aniq maqsad, qabul qiluvchi shaxs va faylning SHA-256 xeshi ko‘rsatilgan holda rasmiylashtiriladi.

### 10-Savol: Laboratoriya modelidan ishlab chiqarish (Enterprise) muhitiga o‘tishda qaysi ikki muhim imkoniyatni birinchi navbatda qo‘shgan bo‘lardingiz?
**Javob:**
1. **NAC uchun:** Port sathidagi **IEEE 802.1X / RADIUS (EAP-TLS)** autentifikatsiyasini va dinamik VLAN boshqaruvini joriy etish (qurilmani MAC/IP bilan emas, unikal xavfsiz sertifikat bilan aniqlash).
2. **DLP uchun:** Shifrlangan **HTTPS (TLS/SSL Decryption / Inspection)** trafigini tekshirish imkoniyatini hamda murakkab hujjat formatlarini (PDF, Word, Excel, ZIP arxivlar) ochib, matnini tahlil qiluvchi chuqur parserlarni qo‘shish.

---

## 6-BO‘LIM. Foydali Ilmiy va Texnik Manbalar

1. **Microsoft Purview Data Loss Prevention Documentation:**
   - *Learn about Data Loss Prevention:* [Microsoft Learn DLP Guide](https://learn.microsoft.com/en-us/purview/dlp-learn-about-dlp)
   - *DLP Policy Reference and Conditions:* [Microsoft Learn DLP Reference](https://learn.microsoft.com/en-us/purview/dlp-policy-reference)
2. **PacketFence Open Source NAC:**
   - *Network Access Control Architecture, Posture Assessment and Captive Portals:* [PacketFence Documentation](https://www.packetfence.com/doc/PacketFence_Administration_Guide.html)
3. **Netfilter / nftables Rasmiy Hujjatlari:**
   - *nft(8) Manual Page & Set Infrastructure:* [Netfilter Project](https://netfilter.org/projects/nftables/manpage.html)
   - *Quick Reference: nftables framework:* [nftables wiki](https://wiki.nftables.org/)
4. **NIST Xavfsizlik Standartlari (Zero Trust & Access Control):**
   - *NIST SP 800-207: Zero Trust Architecture (ZTA):* [NIST Publications](https://csrc.nist.gov/publications/detail/sp/800-207/final)
   - *NIST SP 800-162: Guide to Attribute Based Access Control (ABAC):* [NIST ABAC Guide](https://csrc.nist.gov/publications/detail/sp/800-162/final)
5. **IEEE Standartlari va RFC Hujjatlari:**
   - *IEEE 802.1X-2020: Port-Based Network Access Control:* [IEEE Xplore](https://standards.ieee.org/)
   - *RFC 3580: IEEE 802.1X Remote Authentication Dial In User Service (RADIUS) Usage Guidelines.*
6. **Python 3 Standard Library:**
   - *http.server — HTTP Servers implementation:* [Python 3.12 Documentation](https://docs.python.org/3/library/http.server.html)
   - *hashlib — Secure hashes and message digests:* [Python hashlib](https://docs.python.org/3/library/hashlib.html)

---
*Ushbu qo‘llanma A10 amaliy mashg‘ulotining barcha talablariga to‘liq javob beradi va mustaqil laboratoriya ishini bajarish hamda yuqori ballga himoya qilish uchun mo‘ljallangan.*
