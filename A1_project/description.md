# A1: Tarmoq aktivlari, tahdidlar va risklarni tahlil qilish
**Fan:** Tarmoq xavfsizligi (5-semestr) • **Variant:** 1-variant  
**Mavzu:** Tayyor VMware obrazlari asosida aktivlar inventari va risk reyestrini tuzish  
**Muhit:** **Ubuntu Linux Host** $\rightarrow$ **VMware Workstation for Linux** $\rightarrow$ **2 ta Ubuntu Server VM**  
*(Shaxsiy lokal Wi-Fi / To‘g‘ridan-to‘g‘ri internet, universitet proksisisiz)*

> [!IMPORTANT]
> **LABORATORIYA ISHINING AMALDAGI ISHCHI MUHITI:**  
> Sizning asosiy tizimingiz (Host) — **Ubuntu Linux Desktop** hisoblanadi.  
> * **Host tizimi:** Ubuntu Desktop (Linux)  
> * **Virtualizatsiya vositasi:** VMware Workstation for Linux  
> * **Virtual mashinalar (Mehmon VMlar):** 2 ta mustaqil Ubuntu Server 24.04 VM (`SEC-ANALYST` va `SRV-1`)  
> * **Internet:** Shaxsiy lokal Wi-Fi (Host Ubuntuga ulangan internet NAT orqali VMlarga beriladi). Universitet proksisi (`172.18.1.1:2500`) yoki Windows SMB serveri (`\\172.18...`) **ishlatilmaydi**!  
> * **Fayllar boshqaruvi:** Windows disklari (`D:\...`) emas, Linux standart kataloglari (`~/SEC-LAB/`) va Linux arxivatori (`p7zip-full`) qo‘llaniladi.

---

## MUNDARIJA
1. [1-BO‘LIM: Mavzuning maqsadi, vazifalari va kutiladigan natijalar](#1-bolim-mavzuning-maqsadi-vazifalari-va-kutiladigan-natijalar)
2. [2-BO‘LIM: Mavzular bo‘yicha to‘liq nazariy va texnik qo‘llanma](#2-bolim-mavzular-boyicha-toliq-nazariy-va-texnik-qollanma)
   - [2.1. Axborot xavfsizligining tayanch tushunchalari](#21-axborot-xavfsizligining-tayanch-tushunchalari)
   - [2.2. CIA triadasi va aktiv qiymati](#22-cia-triadasi-va-aktiv-qiymati)
   - [2.3. Riskni sonli baholash metodologiyasi (R = E × T)](#23-riskni-sonli-baholash-metodologiyasi-r--e--t)
   - [2.4. Boshlang‘ich risk va Prognoz qoldiq risk (Residual Risk)](#24-boshlangich-risk-va-prognoz-qoldiq-risk-residual-risk)
   - [2.5. Linux muhitida virtual tarmoq arxitekturasi: vmnet8 vs vmnet2](#25-linux-muhitida-virtual-tarmoq-arxitekturasi-vmnet8-vs-vmnet2)
   - [2.6. Shaxsiy Wi-Fi sharoitida to‘g‘ridan-to‘g‘ri ulanish (Proksisiz)](#26-shaxsiy-wi-fi-sharoitida-togridan-togri-ulanish-proksisiz)
   - [2.7. Skanerlash va audit vositalari: Nmap va ss o‘rtasidagi farq](#27-skanerlash-va-audit-vositalari-nmap-va-ss-ortasidagi-farq)
   - [2.8. Tizim xizmatlari va xavfsizlik nazoratlari (SSH, Nginx, UFW, Loglar)](#28-tizim-xizmatlari-va-xavfsizlik-nazoratlari-ssh-nginx-ufw-loglar)
3. [3-BO‘LIM: Mashqni bajarish ketma-ketligi va Barcha CLI buyruqlarining batafsil tahlili](#3-bolim-mashqni-bajarish-ketma-ketligi-va-barcha-cli-buyruqlarining-batafsil-tahlili)
   - [0-Bosqich: Host Ubuntuda obrazlarni tayyorlash va 2 ta mustaqil VM yaratish](#0-bosqich-host-ubuntuda-obrazlarni-tayyorlash-va-2-ta-mustaqil-vm-yaratish)
   - [1-Bosqich: VMware for Linux-da virtual tarmoqlarni (vmnet2 va vmnet8) sozlash](#1-bosqich-vmware-for-linux-da-virtual-tarmoqlarni-vmnet2-va-vmnet8-sozlash)
   - [2-Bosqich: Server VMlarda paketlarni to‘g‘ridan-to‘g‘ri (proksisiz) o‘rnatish](#2-bosqich-server-vmlarda-paketlarni-togridan-togri-proksisiz-ornatish)
   - [3-Bosqich: NATni uzish va A1-base snapshotlarini yaratish](#3-bosqich-natni-uzish-va-a1-base-snapshotlarini-yaratish)
   - [4-Bosqich: SEC-LAB tarmog‘ida statik IP berish va o‘zaro aloqani tekshirish](#4-bosqich-sec-lab-tarmogida-statik-ip-berish-va-ozaro-aloqani-tekshirish)
   - [5-Bosqich: SRV-1 da lokal aktivlar inventarizatsiyasini o‘tkazish](#5-bosqich-srv-1-da-lokal-aktivlar-inventarizatsiyasini-otkazish)
   - [6-Bosqich: SEC-ANALYST dan tarmoq va xizmatlarni skanerlash (Nmap)](#6-bosqich-sec-analyst-dan-tarmoq-va-xizmatlarni-skanerlash-nmap)
   - [7-Bosqich: Himoya nazoratlarini audit qilish va E1–E8 dalillarini yig‘ish](#7-bosqich-himoya-nazoratlarini-audit-qilish-va-e1e8-dalillarini-yigish)
   - [8-Bosqich: Tahdid modeli va Risk reyestrini shakllantirish](#8-bosqich-tahdid-modeli-va-risk-reyestrini-shakllantirish)
   - [9-Bosqich: Risklarni kamaytirish rejasi va Qoldiq riskni hisoblash](#9-bosqich-risklarni-kamaytirish-rejasi-va-qoldiq-riskni-hisoblash)
   - [10-Bosqich: Yakuniy xulosa tayyorlash va laboratoriyani yakunlash](#10-bosqich-yakuniy-xulosa-tayyorlash-va-laboratoriyani-yakunlash)
   - [To‘ldiriladigan amaliy natija jadvallari (3.1 – 3.5)](#toldiriladigan-amaliy-natija-jadvallari-31--35)
   - [Ko‘p uchraydigan xatolar va ularning yechimlari (Troubleshooting)](#kop-uchraydigan-xatolar-va-ularning-yechimlari-troubleshooting)
   - [O‘qituvchi uchun himoya savollari va mukammal javoblari](#oqituvchi-uchun-himoya-savollari-va-mukammal-javoblari)
4. [4-BO‘LIM: Mavzuga doir manbalar va me’yoriy adabiyotlar](#4-bolim-mavzuga-doir-manbalar-va-meyoriy-adabiyotlar)

---

# 1-BO‘LIM: Mavzuning maqsadi, vazifalari va kutiladigan natijalar

### 1.1. Mashg‘ulot maqsadi
Talaba Linux muhitida to‘liq izolyatsiyalangan virtual laboratoriyada tarmoq aktivlarini aniqlaydi, ularning funksional vazifasi hamda tashkiliy qiymatini tavsiflaydi, ochiq portlar va amaldagi xavfsizlik nazoratlarini texnik dalillar (evidence) asosida qayd etadi; **tahdid – zaiflik – oqibat** sabab-oqibat zanjirini tuzib, ehtimollik va ta’sir mezonlari bo‘yicha risklarni baholaydi hamda ularni minimallashtirish choralarini ishlab chiqadi.

### 1.2. Kutiladigan natijalar
* Host Ubuntu tizimida VMware orqali ikkita mustaqil virtual mashina (**SEC-ANALYST** va **SRV-1**) nusxasini ajratish va to‘g‘ri sozlash;
* Shaxsiy Wi-Fi interneti yordamida Linux paketlarini to‘g‘ridan-to‘g‘ri o‘rnatish, so‘ng xavfsizlik talablariga binoan NAT adapterini uzib, tizimni to‘liq izolyatsiyalash;
* **Aktiv**, **tahdid**, **zaiflik**, **nazorat** va **risk** tushunchalarini amalda farqlay olish;
* Masofaviy tarmoq skaneri (**Nmap**) natijalarini server ichidagi lokal soket ko‘rsatkichlari (**ss**) bilan solishtirib, farqlarni tahlil qilish;
* Kamida **5 ta aktiv** reyestri, **6 ta texnik dalil (E1–E8)**, **6 ta tahdid ssenariysi**, **6 ta asoslangan risk** reyestrini tuzish hamda eng xavfli **3 ta risk uchun prognoz qoldiq riskni** hisoblab chiqish.

### 1.3. Asosiy ko‘rsatkichlar
| Ko‘rsatkich | Qiymati / Tavsifi |
| :--- | :--- |
| **Ajratilgan vaqt** | Asosiy laboratoriya ishi: **80 daqiqa** (Tayyorgarlik 40–60 daqiqa alohida) |
| **Bajarish muhiti** | **Ubuntu Linux (Host)** va **VMware Workstation for Linux** |
| **Virtual mashinalar** | `SEC-ANALYST` va `SRV-1`: `UbuntuServer_24.04_VM.7z` obrazining 2 ta alohida nusxasi (har biri 2 vCPU, 2 GB RAM) |
| **Izolyatsiyalangan tarmoq** | `SEC-LAB`: vmnet2 (Host-only), `192.168.56.0/24`, DHCP o‘chiq, Gateway va DNS yo‘q |
| **Vaqtincha tarmoq** | `vmnet8` (NAT, DHCP yoqilgan) — faqat paket o‘rnatish uchun; skanerlashdan oldin uziladi |
| **Asosiy vositalar** | `iproute2`, `Nmap`, `curl`, `OpenSSH Server`, `Nginx`, `UFW`, `systemd/journalctl`, `7z` |
| **Baholash** | Maksimal **10 ball** |

### 1.4. Xavfsizlik chegaralari (Rules of Engagement)
1. **Faqat SEC-LAB nishon hisoblanadi:** Nmap va barcha audit buyruqlari faqat talabaning o‘z kompyuteridagi `192.168.56.0/24` tarmog‘iga (xususan `192.168.56.20` ga) nisbatan ishlatiladi.
2. **Host Ubuntu va shaxsiy Wi-Fi tarmog‘iga teginish taqiqlanadi:** Shaxsiy Wi-Fi dagi uy routeri (`192.168.1.1` yoki `192.168.0.1`), Host tizimining o‘zi yoki boshqa qurilmalar skanerlanmaydi. Barcha amaliyot faqat `vmnet2` virtual segmentida o‘tishi shart!

---

# 2-BO‘LIM: Mavzular bo‘yicha to‘liq nazariy va texnik qo‘llanma

---

### 2.1. Axborot xavfsizligining tayanch tushunchalari
Axborot xavfsizligi auditi va risklarni boshqarishda (ISO/IEC 27005, NIST SP 800-30) quyidagi 5 ta tushuncha o‘rtasidagi farqni aniq bilish shart:

```
+--------+       Tahdid soladi       +--------+
| Tahdid | ------------------------> | Aktiv  |
+--------+                           +--------+
    |                                    ^
    | Foydalanadi                        | Himoya qiladi
    v                                    |
+---------+       Keltirib chiqaradi  +---------+
| Zaiflik | ----------------------->  | Nazorat |
+---------+                           +---------+
    |
    v
+--------+ (Zarar yetish ehtimoli va oqibati)
|  Risk  |
+--------+
```

1. **Aktiv (Asset):** Tashkilot faoliyati uchun moddiy yoki nomoddiy qiymatga ega bo‘lgan har qanday resurs yoki jarayon.
   * *A1 misoli:* `SRV-1` virtual serveri, uning operatsion tizimi, SSH masofaviy boshqaruv kanali, Nginx veb-xizmati, ichki me’yoriy hujjatlar (veb-kontent), tizim loglari.
2. **Tahdid (Threat):** Aktivga, uning maxfiyligi, yaxlitligi yoki mavjudligiga zarar yetkazishi mumkin bo‘lgan potensial sabab, shaxs yoki hodisa.
   * *A1 misoli:* Tarmoqda buzilgan ichki ishchi kompyuter (compromised host), ruxsatsiz foydalanuvchi (insider threat), tizim administratorining konfiguratsiya xatosi, dasturiy ta’minotdagi nosozlik.
3. **Zaiflik (Vulnerability):** Tahdid manbai tomonidan ekspluatatsiya qilinishi (foydalanilishi) mumkin bo‘lgan xavfsizlik tizimidagi kamchilik, xato yoki bo‘shliq.
   * *A1 misoli:* Xizmatning barcha interfeyslarda keraksiz ochiq turishi, SSH da kuchsiz parol autentifikatsiyasining yoqilganligi, shifrlanmagan HTTP protokoli orqali maxfiy ma’lumot uzatilishi, yangilanmagan dasturiy paketlar.
4. **Nazorat (Control / Safeguard):** Riskni bartaraf etadigan, kamaytiradigan yoki aniqlaydigan texnik, tashkiliy yoki ma’muriy xavfsizlik chorasi.
   * *A1 misoli:* Tarmoqni segmentatsiyalash (Host-only), xavfsizlik devori (UFW), SSH kalitli autentifikatsiyasi (Public Key Auth), parol bilan kirishni taqiqlash, xizmat jurnallarini yuritish (Logging).
5. **Risk (Risk):** Tahdid manbai mavjud zaiflikdan muvaffaqiyatli foydalanishi oqibatida aktivga yetkazilishi mumkin bo‘lgan zararning yuzaga kelish **ehtimolligi** va uning **ta’siri** kombinatsiyasi.
   * *A1 misoli:* Zaif parol sababli ruxsatsiz shaxsning SSH orqali SRV-1 boshqaruvini to‘liq qo‘lga kiritishi (Root Compromise).

---

### 2.2. CIA triadasi va aktiv qiymati
Har qanday axborot aktivi xavfsizlikning uchta klassik ustuni (CIA triadasi) bo‘yicha baholanadi:
* **C — Confidentiality (Maxfiylik):** Axborotning faqat vakolatli shaxslar uchun ochiqligi, ruxsatsiz shaxslardan yashirinligi.
* **I — Integrity (Yaxlitlik):** Ma’lumotlar va tizim konfiguratsiyasining to‘g‘riligi, aniqligi va ruxsatsiz o‘zgartirishlardan himoyalanganligi.
* **A — Availability (Mavjudlik):** Vakolatli foydalanuvchilar talab qilgan vaqtda axborot va xizmatlarning uzluksiz ishlab turishi.

Har bir mezon **1 dan 5 gacha** bo‘lgan ball bilan baholanadi.  
**Aktivning umumiy ustuvorligi (Criticality):** $\max(C, I, A)$ formulasi orqali aniqlanadi. Ya’ni, agar aktiv uchun maxfiylik 2, yaxlitlik 3, lekin mavjudlik 5 bo‘lsa, ushbu aktivning ustuvorligi **5 (Kritik)** deb olinadi.

---

### 2.3. Riskni sonli baholash metodologiyasi (R = E × T)
Laboratoriya ishida risk darajasi yarim miqdoriy usulda baholanadi:
$$\mathbf{R = E \times T}$$
* **E (Ehtimollik / Likelihood):** Tahdidning amalga oshish imkoniyati (1 dan 5 gacha).
* **T (Ta’sir / Impact):** Voqea sodir bo‘lganda yetadigan zarar darajasi (1 dan 5 gacha).
* **R (Risk balli):** 1 dan 25 gacha bo‘lgan ko‘rsatkich.

#### Ehtimollik (E) shkalasi:
| Ball  | Ehtimollik darajasi               | Tavsifi va baholash mezoni                                                                         |
| :---: | :-------------------------------- | :------------------------------------------------------------------------------------------------- |
| **1** | Juda kam (Rare)                   | Faqat favqulodda sharoitda; kuchli va ko‘p bosqichli nazoratlar mavjud.                            |
| **2** | Kam (Unlikely)                    | Nazariy jihatdan mumkin, lekin amalga oshirish uchun murakkab shartlar talab etiladi.              |
| **3** | Mumkin (Possible)                 | Ichki tarmoq sharoitida to‘liq real ssenariy; o‘rtacha malakali tajovuzkor amalga oshira oladi.    |
| **4** | Yuqori (Likely)                   | Zaiflik ochiq ko‘rinib turibdi, himoya nazorati yo‘q yoki o‘chirilgan, ekspluatatsiya qilish oson. |
| **5** | Deyarli muqarrar (Almost Certain) | Tizim doimiy xavf ostida, avtomatlashtirilgan hujumlar kutiladi.                                   |

#### Ta’sir (T) shkalasi:
| Ball | Ta’sir darajasi | CIA va tashkilot faoliyatiga oqibati |
| :---: | :--- | :--- |
| **1** | Arzimas (Insignificant) | Sezilarsiz to‘xtash; maxfiy ma’lumotlar oshkor bo‘lmaydi. |
| **2** | Kichik (Minor) | Qisqa muddatli lokal noqulaylik; ma’lumot yo‘qotilmaydi, tez tiklanadi. |
| **3** | Sezilarli (Moderate) | Bitta xizmat yoki bo‘lim faoliyati to‘xtaydi; cheklangan ma’lumot zarar ko‘radi. |
| **4** | Katta (Major) | Muhim xizmatlar ishdan chiqadi; maxfiy yoki moliyaviy ma’lumotlar jiddiy buziladi. |
| **5** | Kritik (Catastrophic) | Tizim boshqaruvi butunlay yo‘qotiladi, ommaviy ma’lumotlar sizib chiqadi. |

#### Risk matritsasi va boshqaruv qarorlari:
| Risk balli (R) | Xavf darajasi | Boshqaruv qarori |
| :---: | :---: | :--- |
| **1 – 4** | **Past (Low)** | Risk qabul qilinadi (Accept); rejali monitoring olib boriladi. |
| **5 – 9** | **O‘rta (Medium)** | Mas’ul shaxs va muddat belgilanadi; reja asosida kamaytiriladi. |
| **10 – 16** | **Yuqori (High)** | Yaqin muddatda kechiktirmasdan texnik nazorat choralari joriy etiladi. |
| **17 – 25** | **Kritik (Critical)** | Xizmat faoliyati zudlik bilan cheklanadi; favqulodda choralar ko‘riladi. |

#### Risk bayonotining standart qolipi:
> **"Agar [tahdid manbai] [zaiflik/kamchilik]dan foydalansa, [aktiv]ga [aniq oqibat] yuz beradi va [CIA xususiyati] buziladi."**

---

### 2.4. Boshlang‘ich risk va Prognoz qoldiq risk (Residual Risk)
* **Boshlang‘ich risk (Initial Risk):** Hozirgi laboratoriya muhitida amalda ko‘rilgan dastlabki holatdagi risk.
* **Tavsiya etilgan nazorat:** Riskni pasaytirish uchun ko‘rilishi kerak bo‘lgan texnik yoki tashkiliy yechim (masalan: SSH kalitli kirish, UFW yoqish, HTTPS ga o‘tish).
* **Prognoz qoldiq risk (Residual Risk):** Tavsiya etilgan chora joriy etilgandan **keyin** qoladigan xavf darajasi.
  * *Eslatma:* Tavsiya amalda qo‘llanilgach, ehtimollik ($E$) keskin kamayadi ($E=4 \rightarrow 1$), biroq ta’sir ($T$) ko‘p hollarda o‘zgarmaydi, chunki agar hodisa baribir ro‘y bersa, tizimga yetadigan zarar hajmi o‘sha-o‘sha qoladi.

---

### 2.5. Linux muhitida virtual tarmoq arxitekturasi: vmnet8 vs vmnet2
Host Ubuntu tizimingizda VMware Workstation virtual tarmoqlaridan foydalanadi:

```
+-------------------------------------------------------------------------+
|                  HOST TIZIMI: UBUNTU LINUX (Desktop)                    |
|                                                                         |
|  [Wi-Fi Adapteri (wlan0)] ---------> [vmnet8 (NAT Switch):              |
|  (Shaxsiy Wi-Fi / Internet)           DHCP Yoqilgan; Internet uzatadi]  |
|                                                     |                   |
|                 +-----------------------------------+                   |
|                 | (Faqat paket o‘rnatishda ulangan)                     |
|                 v                                                       |
|       +-------------------+                  +--------------------+     |
|       |    vmnet8 (NAT)   |                  | vmnet2 (Host-Only) |     |
|       |  DHCP: Yoqilgan   |                  | Subnet:            |     |
|       |  Adapter 1 (ens33)|                  | 192.168.56.0/24    |     |
|       +---------+---------+                  | DHCP: O‘CHIQ       |     |
|                 |                            | Adapter 2 (ens37)  |     |
|                 |                            +---------+----------+     |
|                 |                                      |                |
|         +-------+-------+                      +-------+-------+        |
|         |               |                      |               |        |
|     (ens33)         (ens33)                (ens37)         (ens37)      |
|  +------+------+ +------+------+        +------+------+ +------+------+ |
|  | SEC-ANALYST | |    SRV-1    |        | SEC-ANALYST | |    SRV-1    | |
|  | (Tahlilchi) | |  (Server)   |        | .56.10      | | .56.20      | |
|  +-------------+ +-------------+        +-------------+ +-------------+ |
|                                                                         |
|  DIQQAT: Paketlar o‘rnatilgach, ens33 (NAT) UZILADI!                    |
|  --> Asosiy tahlil faqat vmnet2 (SEC-LAB) ichida bajariladi!           |
+-------------------------------------------------------------------------+
```

1. **vmnet8 (NAT):** Host Ubuntuning Wi-Fi internetini virtual mashinalarga uzatadi. Unda DHCP yoqilgan bo‘lib, VMlarga internetga chiqish imkonini beradi. Faqat paketlarni o‘rnatishda vaqtincha ishlatiladi.
2. **vmnet2 (Host-only — SEC-LAB):** Maxsus izolyatsiyalangan laboratoriya tarmog‘i (`192.168.56.0/24`).
   * **Talab:** DHCP qat’iyan **o‘chiriladi**.
   * **Xususiyati:** Tarmoqda Gateway va DNS bo‘lmaydi. Bu orqali Nmap skaneri faqat ushbu virtual mashinalar o‘rtasida ishlaydi va Host Ubuntuga yoki shaxsiy Wi-Fi tarmog‘ingizdagi boshqa qurilmalarga ta’sir qilmaydi.

---

### 2.6. Shaxsiy Wi-Fi sharoitida to‘g‘ridan-to‘g‘ri ulanish (Proksisiz)
* Universitet hujjati fakultet ichidagi proksi (`172.18.1.1:2500`) va SMB fayl serveri (`\\172.18...`) uchun mo‘ljallangan.
* Siz esa o‘zingizning Host Ubuntu tizimingizda va shaxsiy Wi-Fi tarmog‘ingizda ishlayapsiz.
* **Xulosa:** Hech qanday proksi (`export http_proxy` yoki `99proxy`) sozlanmaydi! Aks holda paket yuklash ishlamaydi. Shaxsiy internetda to‘g‘ridan-to‘g‘ri `sudo apt update` va `sudo apt install` orqali yuklanadi.

---

### 2.7. Skanerlash va audit vositalari: Nmap va ss o‘rtasidagi farq
* **Nmap (Network Mapper):** Masofaviy tahlilchi nigohi bilan ko‘rilgan natija (Tashqi hujum yuzasi — Attack Surface).
  * `nmap -sn 192.168.56.0/24`: Ping-sweep (Host discovery).
  * `nmap -sT -sV --version-light -p 1-1024 192.168.56.20`: TCP Connect scan (`-sT`) va servis versiyalarini aniqlash (`-sV`).
* **ss (Socket Statistics):** Serverning ichki yadrosi darajasidagi haqiqiy holat.
  * `sudo ss -lntup`: Tizimda qaysi jarayonlar (`-p`), qaysi TCP portlarini (`-t`), qaysi IP manzillarga bog‘langan (bind qilingan) holda kutayotganini (`-l`) chiqaradi.

#### Farq sabablari:
1. **Firewall (UFW):** Xizmat ishlab turgan bo‘lsa ham (`ss` da bor), UFW bloklasa, Nmap uni `filtered` yoki `closed` ko‘rsatadi.
2. **Bind Address:** Agar xizmat faqat `127.0.0.1` ga bog‘langan bo‘lsa, u faqat lokal eshitadi; masofadan Nmap uni ko‘rmaydi. `0.0.0.0` bo‘lsa, barcha interfeyslarda ochiq bo‘ladi.

---

### 2.8. Tizim xizmatlari va xavfsizlik nazoratlari (SSH, Nginx, UFW, Loglar)
1. **OpenSSH Server konfiguratsiyasi (`sshd -T`):**
   * `permitrootlogin`: Agar `yes` bo‘lsa — xavfli zaiflik. Tavsiya: `no`.
   * `passwordauthentication`: Agar `yes` bo‘lsa — brute-force xavfi yuqori. Tavsiya: `no` (faqat SSH kalitlari orqali kirish).
2. **Nginx veb-serveri va ma’lumotlar oshkorligi (Information Disclosure):**
   * `curl -I http://192.168.56.20` qilinganda `Server: nginx/1.24.0 (Ubuntu)` ko‘rinib tursa, bu xizmat versiyasini oshkor qiladi.
3. **UFW (Uncomplicated Firewall):**
   * `sudo ufw status verbose`: O‘rnatilganda odatda `Status: inactive` bo‘ladi, ya’ni tarmoq filtrlash o‘chiq.
4. **Tizim loglari (`journalctl`):**
   * `journalctl -u ssh -n 20`: Ulanishlar tarixi va autentifikatsiya loglari (Audit izi).

---

# 3-BO‘LIM: Mashqni bajarish ketma-ketligi va Barcha CLI buyruqlarining batafsil tahlili

Ushbu bo‘limda har bir qadamdagi **barcha CLI buyruqlari, ularning argumentlari (bayroqlari), tizimda nima vazifa bajarishi va laboratoriyadagi aniq o‘rni** qadamma-qadam tushuntirilgan.

---

### 0-Bosqich: Host Ubuntuda obrazlarni tayyorlash va 2 ta mustaqil VM yaratish
Barcha ishlar sizning asosiy **Host Ubuntu terminalida** bajariladi:

#### 1. Arxivator o‘rnatish va ishchi kataloglarni yaratish:
```bash
sudo apt update
sudo apt install -y p7zip-full
mkdir -p ~/SEC-LAB/extracted ~/SEC-LAB/SEC-ANALYST ~/SEC-LAB/SRV-1
```
* **Buyruqlarning vazifasi va parametrlari tahlili:**
  * `sudo apt update`: Paketlar ombori (repositories) indekslarini yangilaydi, eng so‘nggi dasturlar ro‘yxatini yuklaydi.
  * `sudo apt install -y p7zip-full`: 
    * `install`: Dastur o‘rnatish rejimi.
    * `-y` (*yes*): O‘rnatish jarayonida foydalanuvchidan so‘raladigan barcha tasdiqlashlarga avtomatik ha deb javob beradi.
    * `p7zip-full`: Linuxda `.7z` siqilgan arxiv formatlari bilan ishlash uchun to‘liq utilita.
  * `mkdir -p ...`:
    * `mkdir` (*make directory*): Yangi papka ochish.
    * `-p` (*parents*): Agar ota-papkalar mavjud bo‘lmasa ularni ham bir vaqtda yaratadi va papka allaqachon mavjud bo‘lsa xatolik bermaydi. Bu yerda arxivni ochish (`extracted`) va ikkita VM uchun alohida kataloglar yaratilmoqda.

#### 2. Obraz arxivini ochish:
```bash
7z x UbuntuServer_24.04_VM.7z -o~/SEC-LAB/extracted/
```
* **Buyruq parametrlari tahlili:**
  * `7z`: 7-Zip arxiv boshqaruvchisi.
  * `x` (*extract with full paths*): Arxiv ichidagi barcha fayllar va ichki kataloglar tuzilmasini to‘liq o‘z holicha ochadi.
  * `-o...` (*output directory*): Arxivdan chiqqan fayllar qaysi katalogga tushishini belgilaydi (Diqqat: `-o` va yo‘l o‘rtasida bo‘sh joy qoldirilmaydi).
  * *Vazifasi:* `.vmx` (VM konfiguratsiyasi) va `.vmdk` (virtual disk) fayllarini siqilgan holatdan chiqarish.

#### 3. Ikkita mustaqil VM katalogiga fayllarni nusxalash:
```bash
cp -r ~/SEC-LAB/extracted/* ~/SEC-LAB/SEC-ANALYST/
cp -r ~/SEC-LAB/extracted/* ~/SEC-LAB/SRV-1/
```
* **Buyruq parametrlari tahlili:**
  * `cp` (*copy*): Fayllarni nusxalash.
  * `-r` (*recursive*): Katalog ichidagi barcha fayllar va qism-papkalarni rekursiv ravishda to‘liq ko‘chiradi.
  * `*`: `extracted` ichidagi barcha fayllarni bildiradi.
  * *Vazifasi:* Har bir virtual mashina o‘zining shaxsiy disk fayliga ega bo‘lishi shart. Agar bitta disk fayli ikkala VMga ulansa, fayllar tizimi buziladi (corruption).

#### 4. VMware Workstation for Linux-ni ishga tushirish:
```bash
vmware &
```
* **Buyruq parametrlari tahlili:**
  * `vmware`: VMware Workstation grafik interfeysini ochadi.
  * `&`: Buyruqni fon rejimida (background) ishga tushiradi, natijada terminal band bo‘lib qolmaydi va yangi buyruqlar uchun ochiq turadi.
* **VMware GUI ichidagi harakatlar:**
  * **File → Open** menyusi orqali `~/SEC-LAB/SEC-ANALYST/` ichidagi `.vmx` faylni va `~/SEC-LAB/SRV-1/` ichidagi `.vmx` faylni oching.
  * Nomlarini mos ravishda **SEC-ANALYST** va **SRV-1** deb belgilang.
  * Har biriga **2 vCPU**, **2 GB RAM** ajrating.

#### 5. MAC manzil va Hostname sozlash:
* VMlarni yoqqanda VMware *"I moved it"* yoki *"I copied it"* deb so‘raydi. **"I copied it"** ni tanlang!
  * *Nega:* VMware har bir nusxaga yangi va mustaqil virtual MAC manzil (L2 apparat manzili) generatsiya qiladi. Bu tarmoqda MAC konfliktlarini oldini oladi.
* Host nomlarini berish (Mehmon VM konsolida):
  * **SEC-ANALYST ichida:**
    ```bash
    sudo hostnamectl set-hostname SEC-ANALYST
    ```
  * **SRV-1 ichida:**
    ```bash
    sudo hostnamectl set-hostname SRV-1
    ```
  * *Parametr tahlili:* `hostnamectl set-hostname [nom]` — Linux `systemd-hostnamed` xizmati orqali operatsion tizim nomini o‘zgartiradi va `/etc/hostname` fayliga doimiy yozadi. `exit` qilib qayta kirilganda prompt yangilanadi.

---

### 1-Bosqich: VMware for Linux-da virtual tarmoqlarni (vmnet2 va vmnet8) sozlash
Virtual tarmoqlarni sozlash uchun ikkala VMni to‘g‘ri o‘chiring:
```bash
sudo poweroff
```
* *Vazifasi:* Operatsion tizimni barcha xizmatlar va keshdagi ma’lumotlarni diskka to‘liq yozgan holda xavfsiz o‘chiradi.

#### Host Ubuntuda tarmoq muharririni ochish:
```bash
sudo vmware-netcfg
```
* **Buyruq tahlili:**
  * `sudo`: Administrator (root) huquqi bilan ishga tushirish.
  * `vmware-netcfg`: VMware Workstation for Linux-ning Virtual Network Editor dasturi.
* **Sozlamalar:**
  1. **vmnet8 (NAT):**
     * Turi: **NAT**.
     * *"Connect a host virtual adapter"* va *"Use local DHCP"* yoqilgan bo‘lsin. (Bu sizning shaxsiy Wi-Fi internetingizni VMlarga uzatadi).
  2. **vmnet2 (Host-only — SEC-LAB):**
     * Agar yo‘q bo‘lsa, **Add Network** orqali `vmnet2` ni qo‘shing.
     * Turi: **Host-only**.
     * Subnet IP: `192.168.56.0`, Subnet Mask: `255.255.255.0`.
     * **"Use local DHCP service to distribute IP address to VMs" katagidagi belgini OLIB TASHLANG (DHCP o‘chiq bo‘lsin)!**
     * **Save** tugmasini bosing.
* **Adapterlarni ulash (Har bir VM Settings-da):**
  * **Network Adapter 1:** `NAT` (vmnet8 ga ulanadi).
  * **Network Adapter 2 (Add orqali):** `Custom: Specific virtual network` $\rightarrow$ `vmnet2`.
  * Ikkala adapterda ham *"Connected"* va *"Connect at power on"* yoqilganligini tekshiring.

---

### 2-Bosqich: Server VMlarda paketlarni to‘g‘ridan-to‘g‘ri (proksisiz) o‘rnatish
VMlarni ketma-ket yoqing.

#### Proksini tozalash (Ikkala VMda):
```bash
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY no_proxy NO_PROXY
sudo rm -f /etc/apt/apt.conf.d/99proxy
```
* **Buyruqlar tahlili:**
  * `unset [o‘zgaruvchilar]`: Joriy terminal qobig‘idan (bash) berilgan muhit o‘zgaruvchilarini butunlay o‘chiradi.
  * `sudo rm -f ...`: 
    * `rm` (*remove*): Faylni o‘chirish.
    * `-f` (*force*): Fayl mavjud bo‘lmasa ham xatolik chiqarmasdan o‘chiradi.
    * *Nega kerak:* Agar VMda universitet proksisi yozilgan bo‘lsa, u shaxsiy Wi-Fi da paket yuklashni bloklab qo‘yadi.

#### Aloqa va paket omborini tekshirish:
```bash
ping -c 3 8.8.8.8
sudo apt update
```
* **Buyruqlar tahlili:**
  * `ping -c 3 8.8.8.8`:
    * `ping`: ICMP Echo so‘rov yuborish.
    * `-c 3` (*count*): Cheksiz emas, aynan 3 ta paket yuborib to‘xtaydi.
    * *Vazifasi:* NAT adapteri orqali tashqi internetga 3-darajada aloqa borligini tekshirish.
  * `sudo apt update`: Ubuntu omborlaridagi dasturlar ro‘yxatini internetdan yuklab, lokal indeksni yangilaydi.

#### SEC-ANALYST da paketlarni o‘rnatish:
```bash
sudo apt install -y nmap curl iproute2 iputils-ping
nmap --version | head -n 1
```
* **Buyruqlar tahlili:**
  * `nmap`: Tarmoq skaneri va xavfsizlik auditi vositasi.
  * `curl`: Buyruq satri orqali URL manzillar bilan ishlash (HTTP so‘rovlar) utilitasi.
  * `iproute2`: Zamonaviy Linux tarmoq boshqaruv paketi (`ip`, `ss` buyruqlari shu paketda).
  * `iputils-ping`: Tarmoq aloqasini tekshirish uchun `ping` buyrug‘i.
  * `nmap --version | head -n 1`:
    * `|` (*pipe*): Birinchi buyruq natijasini ikkinchi buyruqqa kirish qilib uzatadi.
    * `head -n 1`: Natijaning faqat eng birinchi satrini (versiyasini) ko‘rsatadi.

#### SRV-1 da xizmatlarni o‘rnatish va faollashtirish:
```bash
sudo apt install -y openssh-server nginx ufw curl iproute2 iputils-ping
sudo systemctl enable --now ssh nginx
systemctl is-active ssh nginx
```
* **Buyruqlar tahlili:**
  * `openssh-server`: SSH protokoli orqali masofadan xavfsiz boshqarish xizmati (`sshd`).
  * `nginx`: Yuqori unumli HTTP/HTTPS veb-server dasturi.
  * `ufw` (*Uncomplicated Firewall*): Linux iptables/nftables xavfsizlik devorini boshqaruvchi qulay interfeys.
  * `sudo systemctl enable --now ssh nginx`:
    * `systemctl`: Linuxda tizim va xizmatlarni (systemd) boshqarish vositasi.
    * `enable`: Xizmatlarni operatsion tizim yoqilganda avtomatik ishga tushadigan (autostart) qiladi.
    * `--now`: Xizmatlarni keyingi yuklanishni kutmasdan, ayni shu soniyada zudlik bilan ishga tushiradi (`start` qiladi).
  * `systemctl is-active ssh nginx`: Xizmatlarning ayni paytda ishlab turganini tekshiradi. Har ikkisi uchun `active` chiqishi shart.

#### Lokal HTTP xizmatini tekshirish (SRV-1 da):
```bash
curl -I http://127.0.0.1
```
* **Buyruq tahlili:**
  * `curl`: HTTP mijoz.
  * `-I` (*head request*): Sahifaning to‘liq HTML kodini emas, faqat server sarlavhalarini (HTTP headers) yuklaydi.
  * `http://127.0.0.1`: Lokal serverning o‘ziga murojaat qilish (loopback). Nginx dan `HTTP/1.1 200 OK` javobi kelishi kerak.

---

### 3-Bosqich: NATni uzish va A1-base snapshotlarini yaratish
Laboratoriya shartiga binoan, tashqi internet va paket yuklash ishlari tugagach, tarmoq to‘liq izolyatsiya qilinishi shart!

1. VMware oynasida har ikkala VM uchun **VM → Settings → Network Adapter 1 (NAT)** sozlamasiga kiring:
   * **"Connected"** belgisini olib tashlang;
   * **"Connect at power on"** belgisini olib tashlang.
   *(Faqat vmnet2 ga ulangan Network Adapter 2 ulangan qoladi).*
2. Virtual mashinalarni to‘g‘ri o‘chiring:
   ```bash
   sudo poweroff
   ```
3. **Snapshot yaratish:**
   VMware oynasida har bir VM uchun: **VM → Snapshot → Take Snapshot...**
   * Nomi: `A1-base`
   * Izoh: `"Paketlar o‘rnatilgan, NAT uzilgan, laboratoriya IP hali berilmagan"`
   * *Nega kerak:* Agar laboratoriya davomida biror xatolik qilib qo‘ysangiz yoki qayta bajarish kerak bo‘lsa, dasturlarni qayta o‘rnatib o‘tirmasdan, bir soniyada ushbu toza nuqtaga qaytish imkonini beradi.
4. Ikkala VMni qayta yoqing.

---

### 4-Bosqich: SEC-LAB tarmog‘ida statik IP berish va o‘zaro aloqani tekshirish
VMlar yoqilgach, avval laboratoriya interfeysining nomini aniqlaymiz:
```bash
ip -br link
```
* **Buyruq tahlili:**
  * `ip`: Tarmoq sozlamalarini ko‘rish va o‘zgartirish utilitasi.
  * `-br` (*brief*): Ma’lumotlarni ixcham, bitta satrda (jadval ko‘rinishida) chiqaradi.
  * `link`: 2-darajali (L2) tarmoq interfeyslari (adapterlar) va ularning MAC manzillarini ko‘rsatadi.
  * *Natija:* Qaysi interfeys laboratoriya adapteri ekanini uning MAC manzili orqali bilib olasiz (odatda `ens37` yoki `ens38`). Quyida `LAB_IF=ens37` misolida keltiriladi.

#### SEC-ANALYST da statik IP kiritish:
```bash
LAB_IF=ens37
sudo ip link set "$LAB_IF" up
sudo ip addr flush dev "$LAB_IF"
sudo ip addr add 192.168.56.10/24 dev "$LAB_IF"
ip -br addr show "$LAB_IF"
```
* **Buyruqlar tahlili:**
  * `LAB_IF=ens37`: Interfeys nomini o‘zgaruvchiga yuklaymiz.
  * `sudo ip link set "$LAB_IF" up`: Interfeysni dasturiy jihatdan faollashtiradi (yoqadi).
  * `sudo ip addr flush dev "$LAB_IF"`: 
    * `flush dev`: Ushbu interfeysda avvaldan qolib ketgan barcha eski IP manzillarni tozalaydi. Bu IP to‘qnashuvini oldini oladi.
  * `sudo ip addr add 192.168.56.10/24 dev "$LAB_IF"`:
    * `addr add`: Yangi IP manzil biriktiradi.
    * `192.168.56.10/24`: SEC-ANALYST ning statik IP si va `/24` (maska `255.255.255.0`).
  * `ip -br addr show "$LAB_IF"`: Berilgan IP to‘g‘ri o‘tirganini ixcham ko‘rsatadi.

#### SRV-1 da statik IP kiritish:
```bash
LAB_IF=ens37
sudo ip link set "$LAB_IF" up
sudo ip addr flush dev "$LAB_IF"
sudo ip addr add 192.168.56.20/24 dev "$LAB_IF"
ip -br addr show "$LAB_IF"
systemctl is-active ssh nginx
```
* *Vazifasi:* Serverga `192.168.56.20/24` manzilini beradi va xizmatlar ishlab turganini qayta tekshiradi.

#### O‘zaro aloqani SEC-ANALYST dan tekshirish:
```bash
ping -c 3 192.168.56.20
ip route get 192.168.56.20
curl -I --max-time 5 http://192.168.56.20
```
* **Buyruqlar tahlili:**
  * `ping -c 3 192.168.56.20`: Tahlilchidan serverga L3 darajasida paket borayotganini va qaytayotganini (RTT vaqtini) ko‘rsatadi.
  * `ip route get 192.168.56.20`: 
    * `route get`: Server manziliga paketlar qaysi interfeys orqali chiqayotganini ko‘rsatadi.
    * *Kutiladigan natija:* `192.168.56.20 dev ens37 src 192.168.56.10` — ya’ni to‘g‘ridan-to‘g‘ri laboratoriya interfeysi orqali yo‘nalgan bo‘lishi shart.
  * `curl -I --max-time 5 http://192.168.56.20`:
    * `--max-time 5`: Agar server javob bermasa, cheksiz kutmasdan 5 soniyada to‘xtatadi.
    * *Vazifasi:* Tarmoq orqali 80-portdagi Nginx veb-serveridan `HTTP/1.1 200 OK` javobini olish.

---

### 5-Bosqich: SRV-1 da lokal aktivlar inventarizatsiyasini o‘tkazish
SRV-1 konsolida quyidagi buyruqlar bajariladi:

```bash
hostnamectl
ip -br addr
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS
free -h
sudo ss -lntup
systemctl --type=service --state=running --no-pager
dpkg-query -W openssh-server nginx ufw
```
* **Buyruqlarning vazifasi va parametrlari tahlili:**
  * `hostnamectl`: Serverning host nomi, arxitekturasi, operatsion tizim distributivi va Linux yadro (kernel) versiyasini ko‘rsatadi (A-01 aktivi dalili).
  * `ip -br addr`: Serverdagi barcha tarmoq interfeyslari va ularning amaldagi IP manzillarini tekshiradi.
  * `lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS`:
    * `lsblk` (*list block devices*): Tizimdagi qattiq disklar va bo‘limlarni ko‘rsatadi.
    * `-o` (*output*): Faqat kerakli ustunlarni chiqaradi (nomi, hajmi, turi, ulanish nuqtasi). Veb-kontent va tizim ma’lumotlari qaysi diskda turganini aniqlaydi.
  * `free -h`:
    * `free`: Operativ xotira (RAM) holatini ko‘rsatadi.
    * `-h` (*human-readable*): Baytlarda emas, MB va GB larda qulay chiqaradi. Tizim resurslari yetarliligini baholaydi.
  * `sudo ss -lntup`:
    * `ss` (*socket statistics*): Yadro darajasidagi tarmoq ulanishlari.
    * `-l` (*listening*): Faqat kutayotgan (tinglayotgan) portlarni ko‘rsatadi.
    * `-n` (*numeric*): Port va IP larni nomga aylantirmasdan raqamda ko‘rsatadi (masalan, `http` emas `80`).
    * `-t` (*tcp*): Faqat TCP protokoli soketlarini ko‘rsatadi.
    * `-u` (*udp*): UDP soketlarini ko‘rsatadi.
    * `-p` (*processes*): Har bir portni qaysi dastur (PID va jarayon nomi) ochib turganini ko‘rsatadi.
  * `systemctl --type=service --state=running --no-pager`:
    * `--type=service`: Faqat tizim xizmatlarini filtrlaydi.
    * `--state=running`: Ayni damda ishlab turgan faol servislarni chiqaradi.
    * `--no-pager`: Natijani `less` orqali emas, to‘g‘ridan-to‘g‘ri terminalga to‘liq bosib chiqaradi.
  * `dpkg-query -W openssh-server nginx ufw`:
    * `dpkg-query -W` (*show*): Ko‘rsatilgan paketlarning o‘rnatilgan aniq versiyalarini bir satrda ko‘rsatadi.

---

### 6-Bosqich: SEC-ANALYST dan tarmoq va xizmatlarni skanerlash (Nmap)
SEC-ANALYST terminalida turib tekshiruvlar bajariladi:

```bash
nmap -sn 192.168.56.0/24
nmap -sT -sV --version-light -p 1-1024 192.168.56.20
curl -I --max-time 5 http://192.168.56.20
```
* **Buyruqlarning vazifasi va parametrlari tahlili:**
  * `nmap -sn 192.168.56.0/24` (E1 dalili):
    * `-sn` (*No port scan* / *Ping sweep*): Portlarni tekshirmasdan, butun L2/L3 tarmog‘ida (`192.168.56.0` dan `.255` gacha) qaysi IP manzillarda tirik qurilmalar (hostlar) borligini aniqlaydi (ARP va ICMP so‘rovlar orqali).
    * *Natija:* `192.168.56.10` (tahlilchi), `192.168.56.20` (server) va `192.168.56.1` (Host virtual adapteri) topiladi.
  * `nmap -sT -sV --version-light -p 1-1024 192.168.56.20` (E2 dalili):
    * `-sT` (*TCP Connect scan*): TCP ning to‘liq 3 bosqichli qo‘l berishish (SYN $\rightarrow$ SYN-ACK $\rightarrow$ ACK) mexanizmi orqali port ochiqligini tekshiradi. Bu standart va xavfsiz skanerlash turi bo‘lib, `root` bo‘lmagan oddiy foydalanuvchida ham ishlaydi.
    * `-sV` (*Version detection*): Ochiq portga maxsus zondlar (probes) yuborib, unda ishlayotgan xizmatning aniq dasturiy nomi va versiyasini aniqlaydi.
    * `--version-light`: Versiyani tezkor va yengil aniqlash rejimi (faqat eng asosiy zondlar yuboriladi, tarmoqni ortiqcha yuklamaydi).
    * `-p 1-1024`: Barcha 65535 ta portni emas, aynan eng muhim standart (Well-Known) 1 dan 1024 gacha bo‘lgan portlarni tekshiradi.
    * *Natija:* `22/tcp open ssh (OpenSSH 9.6p1)` va `80/tcp open http (nginx 1.24.0)` ochiq ekani aniqlanadi.
  * `nmap -Pn -sT -sV --version-light -p 22,80 192.168.56.20`:
    * `-Pn` (*Treat all hosts as online -- skip host discovery*): Agar server ping so‘rovlariga javob bermasa, Nmap uni "o‘chgan" deb o‘ylamasligi uchun ping tekshiruvini aylanib o‘tib, to‘g‘ridan-to‘g‘ri portlarni tekshiradi.

---

### 7-Bosqich: Himoya nazoratlarini audit qilish va E1–E8 dalillarini yig‘ish
SRV-1 terminalida quyidagi buyruqlar bajariladi:

```bash
sudo ufw status verbose
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication|maxauthtries)'
sudo ss -lntup
curl -I --max-time 5 http://192.168.56.20
apt list --upgradable 2>/dev/null
sudo journalctl -u ssh -n 20 --no-pager
sudo journalctl -u nginx -n 20 --no-pager
```
* **Buyruqlarning vazifasi va parametrlari tahlili:**
  * `sudo ufw status verbose` (E4 dalili):
    * `status`: Xavfsizlik devori faol yoki nofaolligini ko‘rsatadi.
    * `verbose`: Qo‘shimcha tafsilotlar — birlamchi qoidalar (default incoming/outgoing siyosati) va interfeys profilini ko‘rsatadi.
    * *Kuzatuv:* `Status: inactive` — himoya devori o‘chiq, tarmoq filtrlash mavjud emas.
  * `sudo sshd -T | grep -E ...` (E5 dalili):
    * `sshd -T` (*test mode / extended config*): OpenSSH serverining barcha konfiguratsiya fayllarini tahlil qilib, ayni paytda xotirada ishlab turgan amaldagi haqiqiy xavfsizlik parametrlarini chiqaradi.
    * `grep -E` (*extended regular expressions*): Chiqishdan faqat bizga kerakli 4 ta parametrni qidirib topadi:
      * `permitrootlogin`: `root` foydalanuvchisiga tarmoqdan to‘g‘ridan-to‘g‘ri kirishga ruxsat bormi (`yes` — xavfli).
      * `passwordauthentication`: Oddiy parol bilan kirishga ruxsat bormi (`yes` — brute-force xavfi).
      * `pubkeyauthentication`: Ochiq kalit (public key) orqali kirish yoqilganmi.
      * `maxauthtries`: Bitta sessiyada parolni noto‘g‘ri kiritishning maksimal soni.
  * `curl -I --max-time 5 http://192.168.56.20` (E6 dalili):
    * HTTP sarlavhalarini tekshiradi. Javobdagi `Server: nginx/1.24.0 (Ubuntu)` satri tizim versiyasi oshkor qilinayotganini (Information Disclosure zaifligi) ko‘rsatadi.
  * `apt list --upgradable 2>/dev/null` (E7 dalili):
    * `apt list --upgradable`: Tizimda qaysi paketlarning yangi versiyalari chiqqanini va yangilanishi kerakligini ko‘rsatadi.
    * `2>/dev/null`: Xatolik xabarlarini (stderr) ekranga chiqarmasdan yo‘qotadi, faqat toza ro‘yxatni qoldiradi.
    * *Vazifasi:* Yangilanmagan paketlar orqali ma’lum CVE zaifliklari borligini tekshirish.
  * `sudo journalctl -u ssh -n 20 --no-pager` (E8 dalili):
    * `journalctl`: `systemd-journald` xizmatining binar loglarini o‘qish utilitasi.
    * `-u ssh` (*unit*): Faqat SSH xizmatiga tegishli loglarni saralaydi.
    * `-n 20` (*number of lines*): Eng oxirgi 20 ta log satrini ko‘rsatadi.
    * `--no-pager`: Natijani to‘xtovsiz to‘liq terminalga chiqaradi (skrinshot olish uchun juda qulay).

---

### 8-Bosqich: Tahdid modeli va Risk reyestrini shakllantirish
Yig‘ilgan E1–E8 dalillari asosida kamida **6 ta tahdid ssenariysi** va **6 ta risk** tuziladi.  
Formula: $\mathbf{R = E \times T}$. Har bir risk kamida bitta CIA xususiyatini (Maxfiylik, Yaxlitlik, Mavjudlik) qamrab olishi lozim.

---

### 9-Bosqich: Risklarni kamaytirish rejasi va Qoldiq riskni hisoblash
Aniqlangan risklar orasidan eng xavfli **3 ta risk** tanlab olinadi va ularni pasaytirish choralari, mas’ul shaxs, muddat hamda prognoz qilingan qoldiq risk hisoblanadi.

---

### 10-Bosqich: Yakuniy xulosa tayyorlash va laboratoriyani yakunlash
1. **Xulosa tuzish:** 6–8 ta mazmunli gapdan iborat bo‘lib, eng muhim aktiv, hujum yuzasi, bosh risk va birinchi joriy etiladigan nazorat aks etishi lozim.
2. **Laboratoriyani toza holatga keltirish:**
   * Barcha hisobot fayllari va skrinshotlarni Host Ubuntudagi o‘z papkangizga saqlang (`~/SEC-LAB/hisobot/`).
   * VMlarni `sudo poweroff` bilan o‘chiring.
   * Keyingi safar laboratoriyani qayta ishga tushirish kerak bo‘lsa, `A1-base` snapshotiga qayting.

---

# To‘ldiriladigan amaliy natija jadvallari (3.1 – 3.5)

### 3.1. Aktivlar reyestri
|    ID    | Aktiv nomi                            | Vazifasi va tashkiliy qiymati                                                | Egasi                | Dalil manbasi          | C (1-5) | I (1-5) | A (1-5) | Ustuvorlik $\max(C,I,A)$ |
| :------: | :------------------------------------ | :--------------------------------------------------------------------------- | :------------------- | :--------------------- | :-----: | :-----: | :-----: | :----------------------: |
| **A-01** | **SRV-1 Server apparat/OT**           | Hujjatlar portali va ma’lumotlar bazasini ko‘tarib turuvchi asosiy platforma | Tizim administratori | `hostnamectl`, `lsblk` |    3    |    4    |    5    |      **5 (Kritik)**      |
| **A-02** | **OpenSSH boshqaruv xizmati**         | Serverni masofadan xavfsiz boshqarish va konfiguratsiya qilish kanali        | Tizim administratori | `ss -lntup`, `dpkg`    |    4    |    5    |    4    |      **5 (Kritik)**      |
| **A-03** | **Nginx veb-xizmati**                 | Foydalanuvchilarga ichki me’yoriy hujjatlarni ko‘rsatuvchi HTTP servis       | Veb-administrator    | `systemctl`, `curl`    |    2    |    3    |    4    |      **4 (Yuqori)**      |
| **A-04** | **Ichki me’yoriy hujjatlar**          | Xodimlar uchun mo‘ljallangan korporativ yo‘riqnomalar va ma’lumotlar         | Bo‘lim rahbari       | `/var/www/html`        |    3    |    4    |    3    |      **4 (Yuqori)**      |
| **A-05** | **Autentifikatsiya va tizim loglari** | Tizimga kirish urinishlari va hodisalar jurnali (Audit izi)                  | Xavfsizlik xodimi    | `journalctl`           |    3    |    4    |    2    |      **4 (Yuqori)**      |

---

### 3.2. Tarmoq va xizmatlar bo‘yicha dalillar (E-ID)
| Dalil ID | Buyruq / Manba               | Haqiqiy kuzatuv (Terminal natijasi)                     | Kutilgan me’yoriy holat                          | Moslik tahlili va xavfsizlik xulosasi                                 |
| :------: | :--------------------------- | :------------------------------------------------------ | :----------------------------------------------- | :-------------------------------------------------------------------- |
|  **E1**  | `nmap -sn 192.168.56.0/24`   | `192.168.56.10` va `192.168.56.20` javob berdi          | SEC-ANALYST va SRV-1 ko‘rinishi kerak            | To‘liq mos. Tarmoq izolyatsiyasi to‘g‘ri, begona hostlar yo‘q.        |
|  **E2**  | `nmap -sT -sV 192.168.56.20` | `22/tcp (OpenSSH 9.6p1)`, `80/tcp (nginx 1.24.0)` ochiq | Faqat zarur xizmatlar ochiq bo‘lishi lozim       | Hujum yuzasida 2 ta ochiq xizmat mavjud; versiyalar aniqlandi.        |
|  **E3**  | `sudo ss -lntup` (SRV-1)     | `0.0.0.0:22` va `0.0.0.0:80` da listen holatida         | Xizmatlar faqat kerakli IP larda eshitishi kerak | Mos. Xizmatlar barcha interfeyslarda ochiq turgani tasdiqlandi.       |
|  **E4**  | `sudo ufw status verbose`    | `Status: inactive`                                      | Xavfsizlik devori faol bo‘lishi kerak            | **Mos emas.** Tarmoq sathi himoyalanmagan, paketlar to‘siqsiz o‘tadi. |
|  **E5**  | `sudo sshd -T`               | `permitrootlogin yes`, `passwordauthentication yes`     | Root taqiqlangan, faqat kalit bilan kirish kerak | **Mos emas.** Brute-force va ruxsatsiz kirish xavfi juda yuqori.      |
|  **E6**  | `curl -I http://...`         | `HTTP/1.1 200 OK`, `Server: nginx/1.24.0`               | Shifrlangan HTTPS va yashirin sarlavha           | **Mos emas.** Shifrlanmagan HTTP; xizmat versiyasi oshkor qilinmoqda. |
|  **E7**  | `apt list --upgradable`      | Bir nechta paket yangilanish kutmoqda                   | Barcha xavfsizlik patchlari o‘rnatilgan bo‘lishi | Ehtimoliy zaifliklar mavjud; yangilash talab etiladi.                 |
|  **E8**  | `sudo journalctl -u ssh`     | Ulanish urinishlari va loginlar qayd etilmoqda          | Hodisalar to‘liq yozilishi kerak                 | Mos. Hodisalarni tahlil qilish va audit o‘tkazish imkoniyati bor.     |

---

### 3.3. Tahdid ssenariylari
|   T-ID   | Aktiv ID | Tahdid manbai                               | Kirish yo‘li (Vektor)                 | Foydalaniladigan zaiflik                                            | Mumkin bo‘lgan oqibat va buziladigan CIA                                   |
| :------: | :------: | :------------------------------------------ | :------------------------------------ | :------------------------------------------------------------------ | :------------------------------------------------------------------------- |
| **T-01** |   A-02   | Buzilgan ichki kompyuter (Compromised host) | SEC-LAB tarmog‘i orqali 22-port (SSH) | SSH da parolli autentifikatsiya va Root kirish ochiqligi (E5)       | Administrator hisobi buziladi, server to‘liq qo‘lga olinadi. (**C, I, A**) |
| **T-02** |   A-04   | Tarmoqdagi vositachi tajovuzkor (MitM)      | Ichki L2 efir/tranzit trafigi         | Veb-trafik shifrlanmagan HTTP orqali uzatilishi (E6)                | Ichki me’yoriy hujjatlar va sessiyalar yashirin o‘g‘irlanadi. (**C**)      |
| **T-03** |   A-04   | Ruxsatsiz ichki foydalanuvchi               | 80-port (HTTP so‘rovlar)              | UFW xavfsizlik devorining o‘chiqligi, kirish cheklanmagani (E4)     | Huquqi bo‘lmagan shaxslar maxfiy portal sahifalarini o‘qiydi. (**C**)      |
| **T-04** |   A-03   | Zararli dastur yoki tajovuzkor              | 80-port (HTTP Flood / DoS)            | Nginx ulanishlar soni va tezligi cheklanmagani                      | Veb-server resurslari tugab, portal xizmati to‘xtab qoladi. (**A**)        |
| **T-05** |   A-01   | Tashqi/ichki niyatli shaxs                  | Ochiq servis portlari (22, 80)        | Dasturiy paketlar o‘z vaqtida yangilanmagani (E7)                   | Ma’lum ekspluat (CVE) orqali tizim yadrosi shikastlanadi. (**I, A**)       |
| **T-06** |   A-02   | Ichki xodim (brute-force)                   | Masofaviy SSH ulanishi                | Muvaffaqiyatsiz urinishlar soni cheklanmaganligi (`maxauthtries 6`) | Parol tanlash orqaliuxsatsiz tizimga kiril radi. (**C, I**)                |

---

### 3.4. Risk reyestri
|   R-ID   | Aktiv | Aniq risk bayonoti                                                                                                                                  | Bog‘liq dalil | E (1-5) | T (1-5) | R (E×T) | Risk darajasi va ustuvorlik |
| :------: | :---: | :-------------------------------------------------------------------------------------------------------------------------------------------------- | :-----------: | :-----: | :-----: | :-----: | :-------------------------: |
| **R-01** | A-02  | Agar tarmoqdagi tajovuzkor SSH parollarini taxmin qilib hujum qilsa, SRV-1 serveriga root huquqi bilan kiradi va butun boshqaruv yo‘qotiladi.       |    E2, E5     |  **4**  |  **5**  | **20**  |    **Kritik (1-o‘rin)**     |
| **R-02** | A-04  | Agar tarmoqdagi vositachi HTTP trafikni ushlab olsa, uzatilayotgan ichki hujjatlar va cookie-fayllarni tutib oladi va ma’lumotlar sizib chiqadi.    |    E2, E6     |  **4**  |  **3**  | **12**  |    **Yuqori (2-o‘rin)**     |
| **R-03** | A-01  | Agar tajovuzkor ochiq portlarga filtrlarsiz to‘g‘ridan-to‘g‘ri murojaat qilsa, UFW yo‘qligi sababli zaif servislarni bemalol ekspluatatsiya qiladi. |      E4       |  **4**  |  **3**  | **12**  |    **Yuqori (3-o‘rin)**     |
| **R-04** | A-03  | Agar zararli botlar Nginx ga ko‘p sonli so‘rov yuborsa, ulanishlar limiti yo‘qligi sababli portal ishlamay qoladi va mavjudlik buziladi.            |    E2, E3     |  **3**  |  **3**  |  **9**  |     **O‘rta (4-o‘rin)**     |
| **R-05** | A-01  | Agar eski paketlardagi ma’lum zaifliklar ekspluatatsiya qilinsa, server yaxlitligi buzilishi mumkin.                                                |      E7       |  **2**  |  **4**  |  **8**  |     **O‘rta (5-o‘rin)**     |
| **R-06** | A-05  | Agar tizim jurnallari faqat lokal saqlansa, tajovuzkor o‘z izlarini o‘chirib yuborishi va hodisani tekshirish imkonsiz bo‘lishi mumkin.             |      E8       |  **2**  |  **3**  |  **6**  |     **O‘rta (6-o‘rin)**     |

---

### 3.5. Risklarni kamaytirish rejasi va Prognoz qoldiq risk
Eng yuqori 3 ta risk (**R-01, R-02, R-03**) bo‘yicha chora-tadbirlar:

|   R-ID   | Amaldagi nazorat     | Tavsiya etilgan texnik chora                                                                                                                                                          | Mas’ul shaxs va muddat                            |         Prognoz qoldiq risk ($E \times T = R$)          | Qabul qilish mezoni (Tasdiq)                                                       |
| :------: | :------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------ | :-----------------------------------------------------: | :--------------------------------------------------------------------------------- |
| **R-01** | Standart login/parol | 1. `PermitRootLogin no` qilish;<br>2. `PasswordAuthentication no` qilib, faqat Ed25519 kalitli kirishni joriy etish;<br>3. SSH portini faqat tahlilchi IP siga cheklash.              | Tizim administratori<br>*(Muddat: 2 kun)*         | $\mathbf{1 \times 5 = 5}$<br>*(O‘rta darajaga tushadi)* | Kalitsiz va parolli kirish to‘liq bloklanishi; SSH faqat `.10` IP dan ochilishi.   |
| **R-02** | Shifrlanmagan HTTP   | 1. TLS/SSL sertifikatini o‘rnatib, HTTPS ga o‘tish;<br>2. HTTP so‘rovlarni avtomatik HTTPS ga yo‘naltirish (301 Redirect);<br>3. `server_tokens off;` sozlash.                        | Veb-administrator<br>*(Muddat: 3 kun)*            | $\mathbf{1 \times 3 = 3}$<br>*(Past darajaga tushadi)*  | Brauzerda trafik to‘liq TLS 1.3 orqali shifrlanishi, Nginx versiyasi yashirilishi. |
| **R-03** | UFW o‘chiq holatda   | 1. UFW ni yoqish (`ufw enable`);<br>2. Barcha kiruvchi trafigini taqiqlash (`default deny incoming`);<br>3. Faqat 192.168.56.10 dan 22-portga va SEC-LAB dan 80/443 ga ruxsat berish. | Xavfsizlik bo‘yicha muhandis<br>*(Muddat: 1 kun)* | $\mathbf{1 \times 3 = 3}$<br>*(Past darajaga tushadi)*  | `ufw status` da Status: active bo‘lishi va qoidalar qat’iy ishlashi.               |

---

### 3.6. Yakuniy xulosa namunasi (Hisobot uchun 6–8 gap)
> "Mazkur laboratoriya tadqiqotida SRV-1 serverining operatsion tizimi va boshqaruv xizmatlari eng yuqori ustuvorlikka ega bo‘lgan kritik aktivlar sifatida baholandi. Masofaviy Nmap skaneri hamda lokal `ss` tahlili orqali serverda 22 (SSH) va 80 (HTTP) portlari barcha interfeyslar uchun ochiq ekani, ya’ni tashqi hujum yuzasi mavjudligi aniqlandi. UFW xavfsizlik devorining nofaolligi va SSH konfiguratsiyasida parolli autentifikatsiya hamda to‘g‘ridan-to‘g‘ri root kirishining yoqilganligi tizimning asosiy zaifliklari ekani dalillar bilan isbotlandi. Tahlil natijasida eng yuqori xavf darajasiga ega bo‘lgan risk sifatida — parollarni buzish orqali serverning to‘liq qo‘lga kiritilishi (R-01, Ball: 20, Kritik) qayd etildi. Shuningdek, ichki portal trafigining shifrlanmagan HTTP orqali uzatilishi ma’lumotlar maxfiyligini yo‘qotish riskini keltirib chiqarmoqda. Ushbu risklarni zudlik bilan kamaytirish uchun birinchi navbatda UFW xavfsizlik devorini faollashtirish hamda SSH boshqaruvini faqat kalitli autentifikatsiyaga o‘tkazish bo‘yicha texnik nazoratlar joriy etilishi shart."

---

# Ko‘p uchraydigan xatolar va ularning yechimlari (Troubleshooting)

| Muammo / Xatolik belgisi | Sababi | Qadamma-qadam tuzatish yo‘li |
| :--- | :--- | :--- |
| **`sudo apt update` xato beryapti (Connecting to 172.18.1.1 timed out)** | Shaxsiy Wi-Fi da universitet proksisi yozib qo‘yilgan. | 1. Proksi faylini o‘chiring: `sudo rm -f /etc/apt/apt.conf.d/99proxy`<br>2. Terminalda tozalang: `unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY`<br>3. `sudo apt update` ni qayta bajaring. |
| **VMware for Linux ochilmayapti yoki kernel moduli so‘rayapti** | Host Ubuntuda VMware modullari (vmmon, vmnet) kompilyatsiya qilinmagan. | Host Ubuntu terminalida bajaring:<br>`sudo vmware-modconfig --console --install-all` |
| **Internet yo‘q (Ping 8.8.8.8 ishlamayapti)** | vmnet8 NAT adapteri ulanmagan yoki Host Ubuntu tarmog‘i uzilgan. | 1. VM Settings da 1-adapter `NAT` va `Connected` ekanini tekshiring.<br>2. Host Ubuntuning Wi-Fi ga ulanganini tekshiring. |
| **NAT uzilgandan keyin ikki VM bir-birini ko‘rmayapti (Ping yo‘q)** | Ikkala VM bir xil vmnet tarmog‘ida emas yoki statik IP berilmagan. | 1. Ikkala VMda ham 2-adapter aynan `vmnet2` ga ulanganini tekshiring.<br>2. `ip -br addr` orqali `.56.10` va `.56.20` manzillari to‘g‘ri interfeysga biriktirilganini tekshiring.<br>3. Interfeysni yoqing: `sudo ip link set dev ens37 up`. |
| **Nginx `active` turibdi, lekin 80-portga ulanib bo‘lmayapti** | UFW filtrlayotgan bo‘lishi yoki Nginx faqat localhost ga bog‘langan bo‘lishi mumkin. | 1. `sudo ss -lntup \| grep 80` qilib, u `0.0.0.0:80` da eshitayotganini tekshiring.<br>2. `sudo ufw status` orqali firewall qoidalarini ko‘ring. |

---

# O‘qituvchi uchun himoya savollari va mukammal javoblari

1. **Laboratoriya arxitekturasi va Host Ubuntu muhitining o‘ziga xosligi nimada?**
   * *Javob:* Ishchi muhit Host sifatida Ubuntu Linux Desktop tizimida qurilgan. Unda VMware Workstation for Linux dasturi ishlatilib, tayyor obrazdan ikkita mustaqil virtual mashina (`SEC-ANALYST` va `SRV-1`) nusxalari ajratilgan. Paketlarni yuklash vaqtida Host Ubuntuning Wi-Fi interneti `vmnet8` (NAT) orqali to‘g‘ridan-to‘g‘ri beriladi, shaxsiy internetda universitet proksisi ishlatilmaydi. Paketlar o‘rnatilgach, laboratoriya xavfsizligi talabiga ko‘ra NAT adapterlari uzilib, tahlil faqat izolyatsiyalangan `vmnet2` (Host-only — `192.168.56.0/24`) tarmog‘ida o‘tkaziladi.

2. **Aktiv bilan dasturiy komponent o‘rtasidagi farq nima? Nginx va veb-kontentni alohida aktiv deb olish qachon to‘g‘ri?**
   * *Javob:* Dasturiy komponent — bu texnik vosita yoki binar jarayon (Nginx dasturi). Aktiv esa tashkilot uchun bevosita qimmatli resursdir. Agar veb-kontent o‘zida tashkilotning ichki me’yoriy hujjatlari yoki maxfiy ma’lumotlarini saqlasa, uni alohida aktiv deb olish shart. Chunki Nginx to‘xtasa — mavjudlik buziladi, kontent o‘g‘irlansa — maxfiylik va yaxlitlik zarar ko‘radi.

3. **Nmap topgan ochiq port nega o‘z-o‘zidan zaiflik yoki risk hisoblanmaydi?**
   * *Javob:* Ochiq port — bu xizmat ishlashi uchun zarur bo‘lgan tarmoq eshigi (dalil). Masalan, veb-server 80-portsiz ishlay olmaydi. Ochiq port faqat unda ishlayotgan dasturda xavfsizlik kamchiligi (kuchsiz parol, eski versiya) bo‘lgandagina zaiflikka aylanadi. Risk esa shu zaiflik orqali tajovuzkor zarar yetkazishi ehtimolidir.

4. **Nmap natijasi bilan `ss -lntup` natijasi nega farq qilishi mumkin?**
   * *Javob:* Nmap masofadan tarmoq orqali qaraydi, `ss` esa server yadrosi ichidagi holatni ko‘rsatadi. Agar xizmat serverda ishlab turgan bo‘lsa-yu, xavfsizlik devori (UFW) portni bloklasa, Nmap uni `filtered` deb ko‘rsatadi. Yoki xizmat faqat `127.0.0.1` ga bog‘langan bo‘lsa, `ss` uni ko‘radi, ammo tashqi Nmap skaneri ko‘ra olmaydi.

5. **Tahdid, zaiflik va oqibatni bitta risk bayonotida qanday bog‘ladingiz?**
   * *Javob:* Standart qolip asosida: *"Agar [buzilgan ichki host / tahdid manbai] [SSH dagi ochiq parolli autentifikatsiya / zaiflik]dan foydalanib brute-force hujumi uyushtirsa, [SRV-1 boshqaruv xizmati / aktiv]ga [ruxsatsiz root kirish / oqibat] yuz beradi va tizim maxfiyligi hamda yaxlitligi buziladi."*

6. **Ehtimollik ballini 4, ta’sir ballini 3 deb tanlaganingizni qaysi dalil bilan asoslaysiz?**
   * *Javob:* Ehtimollik 4 (Yuqori) olinganiga sabab: E4 dalilida UFW ning o‘chiqligi (`inactive`) va E5 da parolli autentifikatsiyaning ochiqligi ko‘rindi, ya’ni tajovuzkor yo‘lida to‘siq yo‘q. Ta’sir 3 (Sezilarli) olinganiga sabab: bu ichki hujjatlar portali bo‘lib, bir bo‘lim ishiga ta’sir qiladi, ammo butun korxona faoliyatini to‘xtatmaydi.

7. **Boshlang‘ich risk va qoldiq risk o‘rtasidagi farq nima?**
   * *Javob:* Boshlang‘ich risk — tizimning joriy holatidagi o‘rganilgan risk. Qoldiq risk esa tavsiya etilgan qo‘shimcha nazoratlar (masalan, SSH kalitli kirish, UFW yoqish, HTTPS) tatbiq etilgandan keyin saqlanib qoladigan minimal xavf darajasidir.

8. **UFW faol bo‘lsa ham risk nega nolga tushmaydi?**
   * *Javob:* Chunki UFW faqat tarmoq sathi (IP va port) bo‘yicha himoya qiladi. Ochiq qoldirilgan 80 yoki 443 portlari orqali keladigan dasturiy darajadagi hujumlarni (masalan, veb-zaifliklar yoki ekspluatlar) xavfsizlik devori to‘xtata olmaydi. Doimo ma’lum bir qoldiq risk saqlanadi.

9. **`apt list --upgradable` chiqishi ekspluatatsiya qilinadigan zaiflik borligini to‘liq isbotlaydimi?**
   * *Javob:* Yo‘q. Yangilanish borligi dasturda tuzatishlar borligini anglatadi, lekin bu tuzatishlar xavfsizlik zaifligi (CVE) ekani va aynan ushbu laboratoriya sharoitida ishlatilishi mumkinligini isbotlamaydi. Bu qo‘shimcha tekshiruv talab qiluvchi xavf belgisidir.

10. **Bir xil ball olgan ikkita riskdan qaysi birini birinchi kamaytirishni qanday tanlaysiz?**
    * *Javob:* 1) Bog‘liq aktivning ustuvorligi $\max(C, I, A)$ qaysi birida yuqori bo‘lsa, o‘sha birinchi olinadi; 2) Xavfsizlik chorasini joriy etish tezligi va murakkabligi (Quick-win) hisobga olinadi — oson va tez hal qilinadigan chora birinchi tatbiq etiladi.

11. **Nega laboratoriya tahlilidan oldin NAT uziladi?**
    * *Javob:* NAT uzilgach, laboratoriya to‘liq izolyatsiyalangan holatga keladi. Bu talabaning ehtiyotsizlik qilib Host Ubuntuni yoki tashqi shaxsiy Wi-Fi tarmog‘idagi qurilmalarni (router, telefonlar) skanerlab qo‘yishining oldini oladi va tekshiruvning xavfsiz chegarada o‘tishini kafolatlaydi.

12. **`A1-base` snapshotini kim va qaysi bosqichda yaratadi?**
    * *Javob:* `A1-base` snapshotini talabaning o‘zi, paketlar o‘rnatilib, NAT adapteri uzilgandan so‘ng, ammo hali SEC-LAB statik IP manzillari berilmasdan oldingi toza holatda yaratadi.

---

# 4-BO‘LIM: Mavzuga doir manbalar va me’yoriy adabiyotlar

Ushbu amaliy ishda qo‘llanilgan uslubiyat xalqaro standartlar va rasmiy texnik qo‘llanmalarga asoslangan:

### 1. Axborot xavfsizligi va risklarni boshqarish standartlari:
* **ISO/IEC 27005:2022** — *Information security, cybersecurity and privacy protection — Guidance on managing information security risks.* (Axborot xavfsizligi risklarini baholash, reyestr yuritish va kamaytirish bo‘yicha xalqaro standart).
* **NIST Special Publication 800-30 Rev. 1** — *Guide for Conducting Risk Assessments.* National Institute of Standards and Technology. (Tahdidlar, zaifliklar va risklarni tahlil qilish bo‘yicha jahon standarti).
* **NIST SP 800-115** — *Technical Guide to Information Security Testing and Assessment.* (Tarmoqni skanerlash va xavfsizlik auditini o‘tkazish metodologiyasi).

### 2. Tarmoq va xizmatlar xavfsizligi bo‘yicha qo‘llanmalar:
* **Nmap Reference Guide (Official)** — Fyodor / Insecure.org. URL: [https://nmap.org/book/man.html](https://nmap.org/book/man.html) (Host discovery `-sn`, TCP Connect `-sT` va servis aniqlash mexanizmlari).
* **Ubuntu Server Guide: Security & Firewall** — Canonical Ltd. URL: [https://ubuntu.com/server/docs/security-firewall](https://ubuntu.com/server/docs/security-firewall) (UFW qoidalari, paket filtrlash va xavfsizlik devori arxitekturasi).
* **OpenSSH Official Manual & Hardening Guide** — OpenBSD Project. URL: [https://www.openssh.com/manual.html](https://www.openssh.com/manual.html) (`sshd_config` parametrlarini xavfsizlash, kalitli autentifikatsiya).
* **CIS Benchmarks (Center for Internet Security)** — *CIS Ubuntu Linux 22.04/24.04 LTS Benchmark.* (Server operatsion tizimini xavfsiz sozlash bo‘yicha sanoat standarti).

### 3. Tarmoq protokollari va tizim utilitalari:
* **RFC 9110 / RFC 9112** — *HTTP Semantics and HTTP/1.1.* IETF Standard. (HTTP sarlavhalari va ulanish mexanizmlari).
* **iproute2 Suite Documentation** — Linux Foundation. URL: [https://wiki.linuxfoundation.org/networking/iproute2](https://wiki.linuxfoundation.org/networking/iproute2) (`ip link`, `ip addr`, `ss` buyruqlari spetsifikatsiyasi).
