# A5: IDS/IPS Tizimlarini O‘rnatish va Hodisalarni Tahlil Qilish
## Laboratoriya ishi bo‘yicha to‘liq tahlil, vositalar katalogi, CLI buyruqlar arxivi va kelgusi rivojlanish yo‘l xaritasi (Roadmap)

> **Ushbu tahliliy hisobot:** `description.md` va `A5_Tarmoq_xavfsizligi_VMware_proxy.docx` fayllarida belgilangan talablar, amaliy topshiriqlar va olingan natijalar asosida tayyorlandi. Hujjat vazifaning tub mohiyatini yoritadi, qo‘llanilgan barcha texnologiyalar va CLI buyruqlarining texnik maqsadi tahlilini beradi hamda axborot xavfsizligi sohasida mustaqil rivojlanish (self-study) uchun batafsil professional yo‘l xaritasini taqdim etadi.

---

## 1* Vazifani bajarishdan asosiy maqsad

Mazkur laboratoriya mashg‘ulotining asosiy mohiyati shunchaki tayyor dasturni (`suricata`) o‘rnatib qo‘yish yoki terminalda bir nechta `curl` va `nmap` buyruqlarini terishdan iborat emas. **Bosh maqsad — zamonaviy korporativ tarmoq xavfsizligining "ko‘zi va qalqoni" bo‘lgan NIDS (Network Intrusion Detection System) va NIPS (Network Intrusion Prevention System) tizimlarining ishlash mexanizmlarini yadro (kernel), paketlar oqimi va xizmat darajasida chuqur tushunish hamda quyidagi konseptual vazifalarni amalda isbotlashdir:**

```
+-----------------------------------------------------------------------------------------+
|                               TARMOQ XAVFSIZLIGI ARCHITEKTURASI                         |
+-----------------------------------------------------------------------------------------+
                                             |
                   +-------------------------+-------------------------+
                   |                                                   |
                   v                                                   v
      [PASSIV IDS REJIMI (Trafik nusxasi)]            [INLINE IPS REJIMI (Trafik yo'lida)]
      - Texnologiya: AF_PACKET (ring buffer)         - Texnologiya: Netfilter / NFQUEUE
      - O'rni: Out-of-band / TAP / SPAN port          - O'rni: In-line ("bump-in-the-wire")
      - Ta'siri: Tarmoq kechikishiga ta'sirsiz        - Ta'siri: Zararli paketni DROP qiladi
      - Drop qoidasi natijasi: action=allowed         - Drop qoidasi natijasi: action=blocked
      - Mijoz: Sahifa ochiladi (HTTP 200)             - Mijoz: Ulanish uziladi (Timeout / Reset)
      - Zaif tomoni: Hujumni to'xtata olmaydi         - Xavfi: Fail-Closed (tarmoq uzilish xavfi)
```

### 1.1. Passiv IDS va Inline IPS o‘rtasidagi fundamental farqni amaliy isbotlash
Nazariy jihatdan hamma biladigan farqni amaliy paketlar va xizmat javoblari darajasida o‘z ko‘zi bilan ko‘rish:
* **Passiv IDS rejimida (`--af-packet`):** Sensor tarmoq kartasidan o‘tayotgan paketlarning faqat **nusxasini** tinglaydi. Hatto qoidada `drop` buyrug‘i yozilgan taqdirda ham, passiv sensor asl paketni to‘xtata olmaydi — asl paket Linux yadrosi orqali serverga o‘tib ketadi, mijozga veb-sahifa to‘liq yetib boradi (`HTTP/1.1 200 OK`), Suricata esa logda faqat `action=allowed` deb qayd etadi.
* **Inline IPS rejimida (`NFQUEUE`):** Sensor paket tranzit yo‘lining toq o‘rtasida turadi. Linux yadrosi (`nftables`) paketni forward qilishdan to‘xtatib, userspace'dagi Suricata navbatiga beradi. Suricata taqiqlangan paketni ko‘rganda yadroga `NF_DROP` buyrug‘ini beradi. Natijada paket serverga yetib bormasdan jismonan yo‘q qilinadi, mijozda `curl: (28) Operation timed out` xatosi chiqadi, logda esa `action=blocked` muhrlanadi.

### 1.2. Ikki segmentli tarmoq router-sensor arxitekturasini yaratish
Haqiqiy tarmoq muhitini modellashtirish uchun ikkita to‘liq izolyatsiyalangan L2 virtual LAN segmenti barpo etiladi:
* `A5-LAN` (`10.50.10.0/24`) — mijoz va hujumchi kompyuterlari joylashgan ichki segment;
* `A5-SRV` (`10.50.20.0/24`) — himoyalanuvchi serverlar joylashgan xavfsiz segment;
* Ikki segment o‘rtasida **`IDS-GW`** shlyuzi o‘rnatilib, Linux yadrosining paketlarni marshrutlash (`net.ipv4.ip_forward = 1`) va Netfilter filtrlash tizimi birlashtiriladi.

### 1.3. Maxsus signaturalar (qoidalar) yozish va turli tahdidlarni aniqlash
Suricata signatura sintaksisining header (sarlavha) va option (parametrlar) qismlarini to‘g‘ri shakllantirish:
* **Tarmoq skanerlashni aniqlash (L3/L4):** ICMP Echo so‘rovlarini (SID 1000501) va TCP SYN ommaviy port skanerlashni (SID 1000503) aniqlash;
* **Ilova sathidagi tekshiruv (L7 Deep Packet Inspection):** HTTP URI va sarlavhalarini tekshirib, xavfsizlik siyosatiga zid bo‘lgan so‘rovlarni aniqlash (SID 1000502) hamda bloklash (SID 1000504).

### 1.4. Telemetriya va Hodisalar Korrelyatsiyasi (Incident Correlation)
Zamonaviy SIEM tizimlarining negizi bo‘lgan **EVE JSON (`eve.json`)** tuzilmasini o‘rganish, hodisalarni `jq` vositasi yordamida saralash, tahlil qilish hamda **`flow_id`** maydoni orqali bir-biriga bog‘liq bo‘lgan Alert, HTTP va Flow hodisalarini yagona zanjirga tizish.

### 1.5. Alert shovqinini (Noise / Fatigue) boshqarish va Qoidalarni sozlash (Tuning)
Xavfsizlik monitoringi xonalarida (SOC) eng katta muammo — bu son-sanoqsiz ma’nosiz signallar tufayli tahlilchilarning charchashi ("Alert Fatigue"). Ushbu muammoni hal qilish uchun qoidaga **`threshold:type limit`** mexanizmini kiritish va xizmatni to‘xtatmasdan (Live Reload) qoidalarni yangilash ko‘nikmasini egallash.

### 1.6. Dalillarning yaxlitligi (Integrity) va xavfsiz tiklash (Rollback) madaniyati
Audit va ekspertiza talablariga muvofiq, olingan barcha PCAP va log dalillarining **SHA-256** nazorat yig‘indilarini hisoblash hamda tizimni asliga qaytarishda tarmoq qotib qolishining (Fail-Closed) oldini olish tartibiga qat’iy rioya qilish.

---

## 2* Vazifani bajarishda qo‘llanilgan tashqi kutubxonalar va qo‘shimcha tool'lar

Laboratoriya ishi davomida virtualizatsiya, tarmoq marshrutlash, xavfsizlik monitoringi, log tahlili va tahdid simulyatsiyasi uchun quyidagi dasturiy vositalar va kutubxonalardan foydalanildi:

| № | Vosita / Tool nomi | Turi va Qatlami | Amaliy vazifasi va laboratoriyadagi o‘rni |
|---|---|---|---|
| **1** | **VMware Workstation Pro / Player for Linux** | Gipervizor (Type-2) | L2 darajasida tashqi Wi-Fi dan mutlaqo ajratilgan `A5-LAN` va `A5-SRV` LAN segmentlarini qurish hamda 3 ta Ubuntu Server VMni izolyatsiyada ushlab turish. |
| **2** | **Ubuntu Server 24.04 LTS** | Operatsion tizim (Guest OS) | `IDS-GW` (marshrutlovchi-sensor), `LAN-CLIENT` (mijoz/hujumchi) va `WEB-SRV` (veb-server) rollarini bajaruvchi barqaror korporativ Linux platformasi. |
| **3** | **p7zip-full (`7z`)** | Tizim utilitasi | Siqilgan `.7z` virtual mashina obrazini ma’lumotlarni yo‘qotmasdan to‘liq va yaxlit ochib olish. |
| **4** | **Suricata (v7.x / v8.x)** | NIDS / NIPS Dvigateli | OISF tomonidan ishlab chiqilgan, ko‘p oqimli (multi-threaded), L7 darajasidagi chuqur paket tekshiruvini (DPI) amalga oshiruvchi asosiy mudofaa dvigateli. |
| **5** | **libnetfilter_queue & libpcap** | Yadro va Tarmoq kutubxonalari | - `libpcap`: Passiv rejimda (`AF_PACKET`) tarmoq interfeysidan xom paketlarni o‘qish.<br>- `libnetfilter_queue`: Linux Netfilter quyi tizimidan paketlarni userspace'dagi Suricata-ga navbat (`queue`) orqali uzatish va verdict olish. |
| **6** | **Netfilter & nftables (`nft`)** | Yadro paket filtrlash tizimi | Linux yadrosining zamonaviy firewall texnologiyasi. Forward zanjiridagi tranzit paketlarni tutib, Suricata navbatiga (`queue num 0`) yo‘naltirish uchun foydalanildi. |
| **7** | **Linux Kernel sysctl (`net.ipv4.ip_forward`)** | Yadro marshrutlash mexanizmi | `IDS-GW` da ikkita alohida tarmoq kartasi (`enp0s8` va `enp0s9`) o‘rtasida paketlarning tranzit harakatini ta’minlash. |
| **8** | **Netplan & systemd-networkd** | Linux tarmoq konfiguratsiyasi | Har uchala virtual mashinada statik IP manzillar, maskalar va default gateway sozlamalarini deklarativ YAML formatida o‘rnatish. |
| **9** | **jq** | Buyruqlar satri JSON protsessori | `eve.json` faylidagi minglab murakkab JSON yozuvlari ichidan faqat kerakli maydonlarni (`timestamp`, `flow_id`, `signature_id`, `action`) ajratib olish, guruhlash va TSV jadvaliga aylantirish. |
| **10** | **Nmap (Network Mapper)** | Tarmoq skaneri / Xavfsizlik auditi | `LAN-CLIENT` dan server tomon cheklangan TCP SYN port skanerlash (20–90/tcp oralig‘ida) o‘tkazib, SID 1000503 qoidasini sinovdan o‘tkazish. |
| **11** | **cURL (`curl`)** | HTTP/HTTPS tarmog‘i mijozi | L7 HTTP so‘rovlarini shakllantirish, maxsus URI markerlarini (`/a5-alert`, `/blocked`) chaqirish va so‘rov natijasini (exit code) tekshirish. |
| **12** | **iputils-ping (`ping`)** | Tarmoq diagnostika utilitasi | ICMP Echo Request paketlarini yuborish, L3 bog‘lanishni tekshirish va SID 1000501 signaturasini qo‘zg‘atish. |
| **13** | **tcpdump** | Tarmoq snifferi (Packet capture) | Tarmoq interfeysidagi real oqimni ushlab, mustaqil `A5_traffic.pcap` fayliga yozib borish (forensic tahlil uchun). |
| **14** | **Nginx** | Veb-server (HTTP daemon) | `WEB-SRV` mashinasida 80-portda turli xil sinov sahifalarini (`/index.html`, `/test.html`, `/blocked.html`) taqdim etuvchi himoyalanuvchi aktiv. |
| **15** | **iproute2 to‘plami (`ip`, `ss`)** | Linux yadro tarmoq asboblari | Interfeyslar holatini ko‘rish (`ip addr`), marshrutlar jadvalini audit qilish (`ip route`) va ochiq soketlarni tekshirish (`ss -lntup`). |
| **16** | **Coreutils va Signal vositalari (`sha256sum`, `kill -USR2`, `sed`, `tar`)** | Tizim ma’murligi va yaxlitlik | Dalillarning SHA-256 xeshlarini chiqarish, konfiguratsiyalarni tahrirlash (`sed`) va xizmatni to‘xtatmasdan jonli yangilash (`USR2` signali). |

---

## 3* Ketma-ket bajarilgan CLI buyruqlar, ularning parametrlari va maqsadi

Mashg‘ulot jarayonida 0-bosqichdan 11-bosqichgacha bajarilgan har bir buyruqning sintaktik tahlili va nima maqsadda ishlatilganligi quyida to‘liq keltirilgan:

### 0-Bosqich: Host Ubuntuda muhitni tayyorlash va mustaqil VM nusxalarini yaratish
* **1. Arxivator o‘rnatish:**
  ```bash
  sudo apt update && sudo apt install -y p7zip-full
  ```
  * `apt update`: Dasturlar ombori indekslarini yangilash.
  * `p7zip-full`: `.7z` arxivlarini ochish uchun to‘liq utilita.
  * **Maqsadi:** Siqilgan virtual mashina obrazini buzmasdan ochib olish.
* **2. Kataloglar yaratish va arxivni ochish:**
  ```bash
  mkdir -p ~/A5-LAB/extracted ~/A5-LAB/IDS-GW ~/A5-LAB/LAN-CLIENT ~/A5-LAB/WEB-SRV
  7z x UbuntuServer_24.04_VM.7z -o~/A5-LAB/extracted/
  ```
  * `mkdir -p`: Ichma-ich papkalarni xatosiz yaratish.
  * `7z x ... -o...`: Arxiv ichidagi fayllar tuzilmasini ko‘rsatilgan katalogga ochish.
  * **Maqsadi:** Har bir VM uchun alohida ishchi maydon ajratish.
* **3. Disk fayllarini 3 ta alohida VM papkasiga nusxalash:**
  ```bash
  cp -r ~/A5-LAB/extracted/* ~/A5-LAB/IDS-GW/
  cp -r ~/A5-LAB/extracted/* ~/A5-LAB/LAN-CLIENT/
  cp -r ~/A5-LAB/extracted/* ~/A5-LAB/WEB-SRV/
  ```
  * `cp -r`: Barcha `.vmx` va `.vmdk` fayllarini to‘liq nusxalash.
  * **Maqsadi:** Har bir VM o‘zining mustaqil virtual diskiga ega bo‘lishi shart, aks holda bitta diskka bir vaqtda yozish ma’lumotlarni yo‘q qiladi (disk corruption).

---

### 1-Bosqich: VMware Workstation for Linux-da tarmoq segmentlarini sozlash
* **VMware menyusi orqali:**
  * Ikkita maxsus LAN Segment yaratildi: `A5-LAN` va `A5-SRV`.
  * `IDS-GW` ga 2 ta tarmoq kartasi berildi: 1-karta $\rightarrow$ `A5-LAN`, 2-karta $\rightarrow$ `A5-SRV`.
  * `LAN-CLIENT` ga 1 ta tarmoq kartasi berildi $\rightarrow$ `A5-LAN`.
  * `WEB-SRV` ga 1 ta tarmoq kartasi berildi $\rightarrow$ `A5-SRV`.
  * **Maqsadi:** L2 darajasida to‘liq izolyatsiya yaratish va tarmoqlararo barcha trafik faqat `IDS-GW` orqali o‘tishini ta’minlash.

---

### 2-Bosqich: Paketlarni shaxsiy Wi-Fi orqali to‘g‘ridan-to‘g‘ri o‘rnatish
*(Dastlabki vaqtinchalik NAT kartasi orqali har bir VM internetga ulanadi; begona proksisiz)*
* **1. Hostname nomlarini to‘g‘rilash:**
  ```bash
  sudo hostnamectl set-hostname IDS-GW       # IDS-GW da
  sudo hostnamectl set-hostname LAN-CLIENT   # LAN-CLIENT da
  sudo hostnamectl set-hostname WEB-SRV      # WEB-SRV da
  ```
  * **Maqsadi:** Terminalda qaysi mashinada ishlayotganini adashtirib yubormaslik.
* **2. IDS-GW mashinasida Suricata va monitoring paketlarini o‘rnatish:**
  ```bash
  sudo apt update
  sudo apt install -y suricata jq tcpdump iproute2 nftables
  suricata -V
  suricata --build-info | grep -E 'NFQueue|AF_PACKET'
  ```
  * `suricata -V`: O‘rnatilgan Suricata versiyasini tekshirish.
  * `--build-info`: Dvigatelda `NFQueue support: yes` va `AF_PACKET support: yes` borligini tekshirish.
  * **Maqsadi:** Inline IPS va passiv IDS uchun kerakli modullar yadro bilan integratsiyalashganini tasdiqlash.
* **3. LAN-CLIENT mashinasida test va skanerlash vositalarini o‘rnatish:**
  ```bash
  sudo apt install -y curl nmap iputils-ping iproute2
  ```
  * **Maqsadi:** HTTP so‘rovlari, SYN port skanerlash va ICMP paketlarini shakllantirish vositalarini hozirlash.
* **4. WEB-SRV mashinasida Nginx veb-serverini o‘rnatish:**
  ```bash
  sudo apt install -y nginx curl iproute2
  sudo systemctl enable --now nginx
  ```
  * **Maqsadi:** Xavfsizlik siyosatiga nishon bo‘ladigan veb-xizmatni ko‘tarish.

---

### 3-Bosqich: NATni uzish, to‘liq izolyatsiya va A5-base snapshot olish
* **Virtual mashinalar o‘chirilib:**
  * NAT interfeyslari olib tashlanadi yoki uziladi.
  * Har bir VMda VMware orqali **`A5-base`** nomli toza snapshot saqlanadi.
  * **Maqsadi:** Tarmoqni tashqi dunyodan to‘liq ajratish va noxush xatolar yuz berganda bir zumda dastlabki toza holatga qaytish kafolati.

---

### 4-Bosqich: Statik IP adresatsiya va IDS-GW routerini sozlash
* **1. IDS-GW da ikkala tarmoq kartasiga IP biriktirish (`/etc/netplan/01-netcfg.yaml`):**
  ```bash
  sudo netplan apply
  ip -br addr
  ```
  * `enp0s8` (LAN interfeysi): `10.50.10.1/24`
  * `enp0s9` (SRV interfeysi): `10.50.20.1/24`
  * **Maqsadi:** Shlyuzni ikki alohida L3 tarmoqning eshigi (gateway) qilib belgilash.
* **2. IDS-GW da Linux IPv4 Forwardingni yoqish:**
  ```bash
  sudo sysctl -w net.ipv4.ip_forward=1
  echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-a5-router.conf
  ```
  * `sysctl -w ...=1`: Linux yadrosida paketlarni bir kartadan ikkinchisiga o‘tkazishni (routing) darhol yoqish.
  * `tee ...`: Ushbu parametrni tizim qayta yuklanganda ham saqlanib qoladigan qilish.
* **3. LAN-CLIENT va WEB-SRV da statik IP va Gateway sozlash:**
  * `LAN-CLIENT`: IP `10.50.10.10/24`, Gateway: `10.50.10.1`
  * `WEB-SRV`: IP `10.50.20.20/24`, Gateway: `10.50.20.1`
  * **Maqsadi:** Mijoz va server paketlarini faqat `IDS-GW` orqali yuborishga majburlash.

---

### 5-Bosqich: WEB-SRV sahifalari va dastlabki aloqani tekshirish (Baseline)
* **1. Serverda 3 xil sinov sahifasini yaratish:**
  ```bash
  echo "A5 NORMAL PAGE" | sudo tee /var/www/html/index.html
  echo "A5 ALERT MARKER PAGE" | sudo tee /var/www/html/a5-alert
  echo "A5 BLOCKED PAGE — IPS bo'lmasa ko'rinadi" | sudo tee /var/www/html/blocked
  ```
  * **Maqsadi:** Har bir test stsenariysi uchun alohida HTTP javob kontentini tayyorlash.
* **2. LAN-CLIENT dan dastlabki aloqani tekshirish:**
  ```bash
  ping -c 2 10.50.20.20
  curl -sS http://10.50.20.20/
  ```
  * **Maqsadi:** IDS/IPS yoqilishidan oldin tarmoq to‘liq va benuqson ishlayotganini (Baseline) tasdiqlash.

---

### 6-Bosqich: Suricata konfiguratsiyasi va lokal qoidalarni (`a5.rules`) yozish
* **1. Standart servislarni to‘xtatish va zaxiralash:**
  ```bash
  sudo systemctl stop suricata
  sudo systemctl disable suricata
  sudo cp /etc/suricata/suricata.yaml /etc/suricata/suricata.yaml.a5.bak
  ```
  * **Maqsadi:** Tizimli avto-start jarayoni bizning qo‘lda boshqaradigan laboratoriya jarayonimiz bilan to‘qnashmasligini ta’minlash.
* **2. `HOME_NET` o‘zgaruvchisini moslash:**
  ```bash
  sudo sed -i 's|HOME_NET: "\[.*\]"|HOME_NET: "\[10.50.10.0/24,10.50.20.0/24\]"|' /etc/suricata/suricata.yaml
  ```
  * **Maqsadi:** Suricata-ga laboratoriyamizning ikkala segmenti ham o‘zimizning ichki ishonchli tarmog‘imiz ekanini uqtirish.
* **3. Maxsus qoidalar faylini (`/etc/suricata/rules/a5.rules`) yaratish:**
  ```text
  alert icmp 10.50.10.0/24 any -> 10.50.20.0/24 any (msg:"A5 LAB ICMP echo request"; itype:8; classtype:misc-activity; sid:1000501; rev:1;)
  alert http 10.50.10.0/24 any -> 10.50.20.20 80 (msg:"A5 LAB HTTP marker"; flow:established,to_server; http.uri; content:"/a5-alert"; startswith; classtype:web-application-activity; sid:1000502; rev:1;)
  alert tcp 10.50.10.0/24 any -> 10.50.20.20 1:1024 (msg:"A5 LAB possible TCP SYN scan"; flags:S; flow:stateless; threshold:type threshold, track by_src, count 8, seconds 10; classtype:network-scan; sid:1000503; rev:1;)
  drop http 10.50.10.0/24 any -> 10.50.20.20 80 (msg:"A5 LAB IPS blocked URI"; flow:established,to_server; http.uri; content:"/blocked"; startswith; classtype:policy-violation; sid:1000504; rev:1;)
  ```
  * **Maqsadi:** 4 xil tahdid stsenariysi (ICMP, Marker, SYN Scan, Taqiqlangan URI) uchun aniq signaturalar bazasini shakllantirish.
* **4. Qoidalarni sintaktik tekshirish (Test mode):**
  ```bash
  sudo suricata -T -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/a5.rules
  ```
  * `-T` (*Test configuration*): Dvigatelni ishga tushirmasdan, faqat YAML va qoidalar sintaksisini tekshirish.
  * **Maqsadi:** Xatoliklarni (imlo xatosi, nuqta-vergul qolib ketishi) oldindan aniqlash; chiqish kodi `0` bo‘lishi shart.

---

### 7-Bosqich: Passiv IDS rejimini ishga tushirish va T1–T5 testlarini bajarish
* **1. IDS-GW da fonda mustaqil PCAP yozish va Suricata-ni yoqish:**
  ```bash
  sudo timeout 45 tcpdump -Z root -i any -nn -s0 -w ~/A5_traffic.pcap 'host 10.50.20.20' &
  sudo suricata -D --pidfile /run/suricata-a5.pid -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/a5.rules --af-packet=enp0s8
  ```
  * `tcpdump -w ~/A5_traffic.pcap`: Barcha paketlarni asl holicha pcap faylga saqlab borish.
  * `--af-packet=enp0s8`: Suricata-ni `enp0s8` kartasidagi paketlar nusxasini passiv o‘qiydigan rejimda ishga tushirish.
* **2. LAN-CLIENT dan T1–T5 testlarini o‘tkazish:**
  ```bash
  # T1: Normal so'rov (Alert chiqmasligi kerak)
  curl -sS http://10.50.20.20/

  # T2: ICMP probe (SID 1000501 alert beradi)
  ping -c 3 10.50.20.20

  # T3: HTTP marker (SID 1000502 alert beradi)
  curl -sS http://10.50.20.20/a5-alert

  # T4: TCP SYN scan (SID 1000503 alert beradi)
  nmap -sT -Pn -p 20-90 --max-rate 20 10.50.20.20

  # T5: Taqiqlangan sahifa so'rovi (Passiv IDS tekshiruvi!)
  curl --max-time 4 -sS http://10.50.20.20/blocked
  echo "T5 exit-kodi=$?"
  ```
  * **Maqsadi va Tahlili:** T1–T4 testlarining barchasi muvaffaqiyatli o‘tadi va tegishli alertlar yoziladi. Eng muhimi — **T5 testida `curl` serverdan sahifani to‘liq qabul qildi va exit kodi `0` bo‘ldi!** Bu passiv IDS qoidasida `drop` yozilgan bo‘lsa ham, paketni to‘xtata olmasligining amaliy isbotidir.

---

### 8-Bosqich: EVE JSON dan ogohlantirishlarni ajratish va flow_id korrelyatsiyasi
* **1. Chiqqan alertlarni TSV jadval qilib chiqarish:**
  ```bash
  sudo jq -r 'select(.event_type=="alert") | [.timestamp, .flow_id, .src_ip, .dest_ip, .proto, .alert.signature_id, .alert.action, .alert.signature] | @tsv' /var/log/suricata/eve.json | tee ~/A5_ids_alerts.tsv
  ```
  * `jq -r`: Xom (raw) ma’lumotlarni chiqarish.
  * `select(.event_type=="alert")`: Faqat ogohlantirishlarni filtrlash.
  * `@tsv`: Ustunlarni tabulyatsiya bilan chiroyli jadval qilish.
* **2. SID bo‘yicha guruhlash va sanash:**
  ```bash
  sudo jq -s 'map(select(.event_type=="alert")) | group_by(.alert.signature_id) | map({sid: .[0].alert.signature_id, count: length, action: .[0].alert.action})' /var/log/suricata/eve.json
  ```
  * **Maqsadi:** Har bir signatura necha marta ishlagani va unga qanday amal (`allowed`) qo‘llanganini umumlashtirish.
* **3. Yagona `flow_id` bo‘yicha hodisalar zanjirini bog‘lash (Correlation):**
  ```bash
  FLOW_ID=$(sudo jq -r 'select(.event_type=="alert" and .alert.signature_id==1000504) | .flow_id' /var/log/suricata/eve.json | tail -n 1)
  sudo jq --argjson id "$FLOW_ID" 'select(.flow_id==$id) | {timestamp, event_type, http, alert, flow}' /var/log/suricata/eve.json
  ```
  * **Maqsadi:** Aynan bitta `flow_id` ga ega bo‘lgan 3 ta hodisani (`alert`, `http`, `flow`) birlashtirib, bitta so‘rovning butun hayotiy siklini isbotlash.

---

### 9-Bosqich: NFQUEUE asosida inline IPS rejimiga o‘tish va T6–T7 bloklash testlari
* **1. Suricata-ni NFQUEUE rejimida qayta yoqish:**
  ```bash
  sudo kill "$(cat /run/suricata-a5.pid)" && sudo rm -f /run/suricata-a5.pid
  sudo suricata -D --pidfile /run/suricata-a5.pid -c /etc/suricata/suricata.yaml -S /etc/suricata/rules/a5.rules -q 0
  ```
  * `-q 0`: Suricata-ga Netfilter'ning 0-navbatini (`queue num 0`) tinglashni buyurish.
* **2. nftables orqali tranzit paketlarni navbatga yo‘naltirish:**
  ```bash
  sudo nft add table inet a5_ips
  sudo nft 'add chain inet a5_ips forward { type filter hook forward priority 10; policy accept; }'
  sudo nft add rule inet a5_ips forward iifname "enp0s8" oifname "enp0s9" counter queue num 0
  sudo nft add rule inet a5_ips forward iifname "enp0s9" oifname "enp0s8" counter queue num 0
  sudo nft -a list table inet a5_ips
  ```
  * `hook forward priority 10`: Yadro orqali o‘tayotgan har bir tranzit paketni ushlash.
  * `counter queue num 0`: Har bir paketni hisoblab, userspace'dagi Suricata-ga jo‘natish.
* **3. LAN-CLIENT da T6 va T7 IPS testlarini o‘tkazish:**
  ```bash
  # T6: Normal sahifa (Erkin o'tishi shart)
  curl --max-time 4 -sS http://10.50.20.20/
  echo "T6 exit-kodi=$?"

  # T7: Taqiqlangan sahifa (BLOKLANISHI SHART!)
  curl --max-time 4 -sS http://10.50.20.20/blocked
  echo "T7 exit-kodi=$?"
  ```
  * **Natija va Isbot:** T6 muammosiz ochiladi (exit code 0). Ammo **T7 testida sahifa OCHILMADI!** 4 soniyadan keyin `curl: (28) Operation timed out` xatosi chiqdi va exit kodi `28` bo‘ldi.
* **4. Bloklash dalilini EVE logida tasdiqlash:**
  ```bash
  sudo jq -r 'select(.alert.signature_id==1000504) | [.timestamp, .flow_id, .alert.action, .alert.signature] | @tsv' /var/log/suricata/eve.json | tail -n 2
  ```
  * Logda **`action: "blocked"`** deb qayd etilganligi IPS paketni yo‘q qilganining 100% dalilidir.

---

### 10-Bosqich: Alert shovqinini threshold bilan sozlash (Tuning)
* **1. Qoidaga threshold qo‘shish va rev versiyasini oshirish:**
  ```bash
  sudo sed -i 's/itype:8; classtype/itype:8; threshold:type limit, track by_src, count 1, seconds 30; classtype/' /etc/suricata/rules/a5.rules
  sudo sed -i 's/sid:1000501; rev:1;/sid:1000501; rev:2;/' /etc/suricata/rules/a5.rules
  ```
  * `threshold:type limit, track by_src, count 1, seconds 30`: Bitta manbadan 30 soniya ichida qancha ping kelsa ham, faqat 1 martagina ogohlantirish berish.
* **2. Xizmatni to‘xtatmasdan jonli qayta yuklash (Live Reload):**
  ```bash
  sudo kill -USR2 "$(cat /run/suricata-a5.pid)"
  sudo tail -n 10 /var/log/suricata/suricata.log
  ```
  * `kill -USR2`: Suricata jarayoniga qoidalarni qayta o‘qish signalini berish (downtimesiz).
* **3. Tuning samarasini tekshirish:**
  * Mijozdan 5 ta ping yuboriladi: `ping -c 5 10.50.20.20`
  * EVE logida alertlar sanaladi: 5 ta pingga qaramay, **atigi 1 ta yangi alert** qo‘shiladi. Shovqin 80% ga qisqartirildi!

---

### 11-Bosqich: Dalillarni eksport qilish, SHA-256 xeshlash va xavfsiz tiklash (Rollback)
* **1. Dalillar papkasini yig‘ish va SHA-256 hisoblash:**
  ```bash
  mkdir -p ~/A5_hisobot
  cp ~/A5_ids_alerts.tsv ~/A5_eve_ids.json ~/A5_eve_ips.json ~/A5_traffic.pcap ~/A5_hisobot/
  (cd ~/A5_hisobot && find . -maxdepth 1 -type f ! -name A5_SHA256SUMS.txt -exec sha256sum {} + > A5_SHA256SUMS.txt)
  cat ~/A5_hisobot/A5_SHA256SUMS.txt
  ```
  * **Maqsadi:** Forensik dalillarning soxtalashtirilmaganligini kriptografik jihatdan muhrlash.
* **2. Tizimni xavfsiz tiklash (Rollback):**
  ```bash
  # 1-qadam: Avval nftables jadvalini o'chirish (Fail-closed ning oldini olish)
  sudo nft delete table inet a5_ips
  # 2-qadam: So'ngra Suricata-ni to'xtatish
  sudo kill "$(cat /run/suricata-a5.pid)"
  # 3-qadam: Forwardingni o'chirish va asl holatga qaytish
  sudo sysctl -w net.ipv4.ip_forward=0
  ```
  * **O‘ta muhim sabab:** Agar avval Suricata o‘chirilsa, nftables paketlarni bo‘sh navbatga jo‘natishda davom etadi va butun tarmoq trafigi qotib qoladi!

---

## 4* Self-study yo‘l xaritasi (Roadmap): Qaysi darajaga yetganda bularni o‘rganish kerak?

A5 laboratoriya ishi — bu oddiy boshlang‘ich dasturlash yoki umumiy kompyuter savodxonligi emas, balki **Kiberxavfsizlik (Cybersecurity) va Tarmoq muhandisligining O‘rta (Intermediate / Mid-level)** bosqichiga tegishli amaliyotdir.

Mustaqil o‘rganish (self-study) qilayotganingizda, quyidagi yo‘l xaritasiga asoslanib qadam tashlashingiz tavsiya etiladi:

```
+-----------------------------------------------------------------------------------------+
|                    KIBERXAVFSIZLIK VA SOC ANALYST YO'L XARITASI                         |
+-----------------------------------------------------------------------------------------+
                                             |
   [1-BOSQICH: POYDEVOR (Foundation)]        v
   - Linux tizim ma'murligi: Bash, systemd, netplan, permissions
   - Tarmoq asoslari: OSI modeli, TCP/IP, Subnetting (/24, /28), ARP, DNS, DHCP
   - Asosiy utilitalar: ping, traceroute, curl, netcat
                                             |
   [2-BOSQICH: JUNIOR SECURITY / L1 SOC]     v
   - Tarmoq auditi: Nmap (SYN scan, Service detection, NSE skriptlar)
   - Trafik tahlili: Wireshark va tcpdump (PCAP o'qish, TCP bayroqlari)
   - Linux Hardening: UFW/iptables, SSH kalitlari, fail2ban
   - Log tahlili: syslog, journalctl, auth.log
                                             |
   [3-BOSQICH: MID-LEVEL SOC & NIDS/NIPS]    v   <==== [SIZNING HOZIRGI A5 DARANGIZ]
   - Zamonaviy NIDS/NIPS: Suricata, Snort 3, Zeek
   - Linux Netfilter arxitekturasi: nftables, NFQUEUE, raw tables
   - Signatura muhandisligi: Custom qoidalar, Thresholding, Suppression
   - Telemetriya: EVE JSON, jq bilan ma'lumotlarni filtrlash, flow_id korrelyatsiyasi
   - Alert Fatigue va Tuning: False Positive/False Negative larni kamaytirish
                                             |
   [4-BOSQICH: SENIOR DEFENDER / SIEM ENG]   v
   - SIEM tizimlari: Wazuh, Elastic Security (ELK), Splunk, Graylog
   - Shifrlangan trafik tahlili: TLS JA3/JA4 fingerprinting, SNI inspection
   - Dvigatel unumdorligi: AF_PACKET v3, DPDK, eBPF / XDP (10G/40G tarmoqlarda)
   - Threat Hunting & Sigma: MITRE ATT&CK ga xaritalash, IOC (Indicators of Compromise)
```

### Ushbu mavzuni qachon va qaysi darajaga yetganda chuqurlashtirish kerak?
1. **Linux va Tarmoq bo‘yicha poydevor to‘liq mustahkamlangach:**
   * Siz `ip route`, `iptables/nftables`, `systemctl`, `grep`, `awk`, `sed` buyruqlarida qiynalmaydigan bo‘lganingizda;
   * TCP 3-bosqichli qo‘l berishish (Handshake: SYN $\rightarrow$ SYN/ACK $\rightarrow$ ACK) va sessiyani tugatish (FIN, RST) qanday ishlashini yoddan bilganingizda.
2. **Karyera yo‘nalishlari:**
   * **SOC Analyst (L1/L2 — Security Operations Center):** Har kuni minglab alertlar ichidan haqiqiy insidentlarni ajratish uchun EVE JSON va `flow_id` korrelyatsiyasini bilish shart.
   * **Network Security Engineer:** Korxona perimetrida Cisco Firepower, Fortinet yoki pfSense/OPNsense (Suricata bazasida) xavfsizlik devorlarini sozlash uchun.
   * **Blue Team / Incident Responder:** Hujumni tarmoq izlari (PCAP) va signaturalar orqali aniqlash uchun.

---

## 5* Shaxsiy maslahatlarim va professional tavsiyalar

Xavfsizlik muhandisi va tahlilchi sifatida tezroq professional darajaga ko‘tarilishingiz uchun amaliy tavsiyalar:

1. **"Paket simini his eting" — Wireshark va tcpdump dan ajralmang:**
   * Hech qachon faqatgina dasturning yashil yoki qizil xabariga ishonmang. T5 va T7 testlarini o‘tkazayotganda `~/A5_traffic.pcap` faylini Wireshark dasturida ochib ko‘ring. T5 da TCP FIN/ACK qanday almashgani, T7 da esa Suricata paketni yadroda yutib yuborgani sababli mijoz qayta-qayta TCP Retransmission yuborib oxiri qotib qolganini ko‘ring. Bu sizga 10 barobar chuqur tushuncha beradi.
2. **Inline IPS dagi eng xavfli nuqta — "Fail-Closed" va "Availability" balansi:**
   * Real kompaniyalarda xavfsizlikdan ko‘ra biznesning uzluksiz ishlashi (Availability) ko‘pincha ustun turadi. Agar siz noto‘g‘ri yoki xato yozilgan qoidani inline IPS ga qo‘shsangiz, butun boshli bank yoki to‘lov tizimi to‘xtab qolishi mumkin. Shuning uchun oltin qoida: **"Har qanday yangi signatura avval kamida 1–2 hafta passiv IDS rejimida sinovdan o‘tishi, False Positive bermasligi tasdiqlangachgina IPS rejimiga o‘tkazilishi shart!"**
3. **Signatura yozishda False Positive va False Negative balansi:**
   * Agar qoidangiz juda qattiq bo‘lsa (`content:"admin"`), oddiy xodim ham bloklanadi (False Positive). Agar juda yumshoq bo‘lsa, buzg‘unchi o‘tib ketadi (False Negative). Doimo kontekstni toraytiring: `flow:established,to_server; http.uri; startswith;` kabi aniq modifierlardan foydalaning.
4. **Zamonaviy muammo: Shifrlangan trafik (HTTPS/TLS) bilan nima qilamiz?**
   * Bugungi kunda internet trafigining 95% qismi HTTPS (TLS) orqali o‘tadi. Suricata ichidagi `http.uri` faqat shifrlanmagan HTTP da ishlaydi. HTTPS da nima qilish kerak?
     * **TLS Fingerprinting:** Mijozning TLS sarlavhasidagi `tls.sni` (Server Name Indication) va JA3/JA4 xeshlarini tahlil qilish;
     * **SSL/TLS Decryption (Interception):** Korxona shlyuzida proksi orqali trafikni yechib, keyin Suricata-ga berish.
5. **Laboratoriyani kengaytiring — SIEM bilan integratsiya qiling:**
   * Ushbu laboratoriyani yanada qiziqarli qilish uchun `IDS-GW` ga **Wazuh Agent** yoki **Filebeat** o‘rnating va `eve.json` loglarini markaziy **Elasticsearch / Kibana** ga yuboring. Chiroyli grafiklar va dashboardlar orqali real SOC muhitini o‘z kompyuteringizda yarating.
6. **Rezyume (CV) va Portfolio uchun oltin aktiv:**
   * Bajarilgan ushbu laboratoriya ishini (A5) GitHub profilingizga joylang: unda arxitektura diagrammasi, o‘zingiz yozgan signaturalar, `jq` buyruqlari va skrinshotlarni tartibli `Writeup` holida saqlang. Bu intervyularda har qanday diplomdan ko‘ra kuchliroq taassurot qoldiradi.
7. **Tavsiya etiladigan amaliy platformalar va sertifikatlar:**
   * **TryHackMe:** *"Suricata"*, *"Snort"*, *"Network Security Monitoring"* xonalari.
   * **LetsDefend:** Real SOC tahlilchisi simulyatori (alertlarni tahlil qilish).
   * **Sertifikatlar:** CompTIA Security+, CySA+, Cisco CyberOps Associate, Blue Team Level 1 (BTL1).

---

## Yakuniy Xulosa

`A5` laboratoriya mashg‘uloti — zamonaviy tarmoq xavfsizligi arxitekturasining eng muhim amaliy pog‘onalaridan biridir. Ushbu dars orqali talaba passiv kuzatuv (Detection) bilan faol to‘xtatish (Prevention) o‘rtasidagi farqni real tarmoq paketlari darajasida sinab ko‘rdi, Linux yadrosining `nftables` va `NFQUEUE` texnologiyalari orqali xavfsizlik devori bilan NIDS tizimini birlashtirishni o‘rgandi hamda zamonaviy EVE JSON formatidagi telemetriyani tahlil qilish ko‘nikmasini mustahkamladi. Ushbu bilimlar kelgusida korporativ xavfsizlik markazlarida (SOC) muvaffaqiyatli faoliyat yuritish uchun mustahkam poydevor bo‘lib xizmat qiladi.
