# A1: Tarmoq aktivlari, tahdidlar va risklarni tahlil qilish
## Laboratoriya ishi bo‘yicha to‘liq xulosa, vositalar tahlili va yo‘l xaritasi (Roadmap)

> **Ushbu hujjat:** `description.md` va `A1_Tarmoq_xavfsizligi_VMware_proxy.pdf` fayllari asosida bajarilgan ishlarni umumlashtiradi, foydalanilgan barcha vositalar va CLI buyruqlarini tizimlashtiradi hamda mustaqil o‘rganish (self-study) uchun yo‘l xaritasini taqdim etadi.

---

## 1* Vazifani asosiy bajarishdan maqsad

Ushbu laboratoriya ishining tub mohiyati shunchaki konsolda buyruqlarni ko‘r-ko‘rona terish yoki Nmap orqali portlarni skanerlashdan iborat emas. **Asosiy maqsad — professional xavfsizlik mutaxassisi (Security Analyst / Auditor) fikrlash doirasini shakllantirish** va amaliy tizimda quyidagi konseptual vazifalarni to‘liq bajarishdir:

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

1. **Izolyatsiyalangan sinov maydonchasini (Sandbox) to‘g‘ri loyihalash:**
   * Tashqi real tarmoqqa yoki shaxsiy Wi-Fi tizimiga ziyon yetkazmaslik uchun virtualizatsiya (VMware) imkoniyatlaridan foydalangan holda to‘liq yopiq L2 segment (`vmnet2` / `192.168.56.0/24`) yaratish.
   * Paketlarni xavfsiz yuklab olgach, NAT adapterini uzib, tarmoqni to‘liq izolyatsiya qilish madaniyatini shakllantirish.

2. **Axborot xavfsizligining tayanch zanjirini amalda bog‘lash:**
   * **Aktiv (Asset):** Faqat temir yoki dastur emas, balki korxona uchun qiymatga ega bo‘lgan resurs (masalan, veb-serverdagi ichki me’yoriy hujjatlar, SSH boshqaruv kanali, audit loglari).
   * **Tahdid (Threat):** Aktivga putur yetkazishi mumkin bo‘lgan real manba (tarmoqdagi zararlangan kompyuter, ruxsatsiz ichki xodim, administrator xatosi).
   * **Zaiflik (Vulnerability):** Tizimda mavjud kamchilik (parolli SSH kirish, root ruxsati, UFW firewallning o‘chiqligi, shifrlanmagan HTTP).
   * **Nazorat (Control):** Xavfni jilovlovchi vositalar (tarmoq segmentatsiyasi, SSH public key, firewall qoidalari).
   * **Risk (Risk):** Tahdidning zaiflik orqali aktivga yetkazishi mumkin bo‘lgan aniq zarari.

3. **Texnik dalillar (Evidence E1–E8) asosida xulosalar chiqarish:**
   * Quruq gap yoki taxmin bilan emas, balki terminal chiqishlari (`sshd -T`, `ss -lntup`, `nmap`, `ufw status`, `curl -I`) orqali har bir zaiflikni fakt bilan isbotlash.

4. **Xavf darajasini yarim miqdoriy baholash ($R = E \times T$):**
   * Ehtimollik ($E=1..5$) va Ta’sir ($T=1..5$) mezonlari bo‘yicha risk ballarini hisoblash ($R=1..25$), xavflarni ustuvorligi bo‘yicha saralash (Prioritization).

5. **Risklarni minimallashtirish rejasi va Prognoz qoldiq risk (Residual Risk):**
   * Eng xavfli 3 ta risk bo‘yicha aniq texnik choralar belgilash, mas’ul shaxs va muddatni tayinlash hamda chora ko‘rilgandan keyin qoladigan xavf darajasini oldindan prognoz qilish.

---

## 2* Vazifani bajarishda qo‘llanilgan tashqi kutubxonalar va qo‘shimcha tool’lar

Laboratoriya muhitini qurish, boshqarish, tahlil qilish va audit o‘tkazishda quyidagi tizimli vositalar hamda utilitalardan foydalanildi:

| № | Vosita / Tool nomi | Turi va Muhiti | Asosiy vazifasi va laboratoriyadagi o‘rni |
|---|---|---|---|
| **1** | **VMware Workstation** *(Linux / Windows)* | Gipervizor (Type 2 Hypervisor) | Virtual mashinalar (`SEC-ANALYST`, `SRV-1`) arxitekturasini ko‘tarish, vSwitch tarmoqlari (`vmnet8`, `vmnet2`) yaratish, tizim holatlarini saqlash (`A1-base` snapshot). |
| **2** | **vmware-netcfg** *(Virtual Network Editor)* | Tarmoq muharriri (GUI/CLI) | Virtual tarmoq segmentlarini boshqarish, `vmnet2` da DHCP ni majburiy o‘chirish, subnet va maskalarni belgilash. |
| **3** | **p7zip-full (`7z`)** | Tizim utilitasi / Arxivator | Siqilgan `.7z` virtual obraz ichidagi `.vmx` va `.vmdk` disk fayllarini to‘liq, yaxlit holda ochib olish. |
| **4** | **Ubuntu Server 24.04 LTS** | Operatsion tizim (Guest OS) | Tahlilchi va server uchun xizmat qiluvchi korporativ Linux platformasi. |
| **5** | **iproute2 to‘plami (`ip`, `ss`)** | Linux yadro tarmoq to‘plami | - `ip link/addr/route`: Tarmoq interfeyslarini boshqarish, statik IP biriktirish, marshrutlarni tekshirish.<br>- `ss`: Yadro darajasida ochiq soketlar, portlar va ularni ushlab turgan jarayonlar (PID) tahlili. |
| **6** | **iputils-ping (`ping`)** | Tarmoq utilitasi | ICMP Echo protokoli orqali L3 darajasidagi tarmoq bog‘lanishini (Reachability & RTT) tekshirish. |
| **7** | **curl** | Tarmoq/HTTP mijozi | Veb-serverga HTTP/HTTPS so‘rovlar yuborish, server sarlavhalarini (`curl -I`) olish, proksi orqali/proksisiz ulanishni tekshirish. |
| **8** | **Nmap (Network Mapper)** | Tarmoq skaneri va xavfsizlik auditi vositasi | Tarmoqdagi jonli hostlarni topish (`-sn`), TCP portlarni tekshirish (`-sT`), xizmatlar va ularning dasturiy versiyalarini aniqlash (`-sV`). |
| **9** | **OpenSSH Server (`openssh-server`, `sshd`)** | Server xizmati va boshqaruv protokoli | Serverni masofadan boshqarish xizmati. `sshd -T` buyrug‘i orqali uning xavfsizlik konfiguratsiyasi (Root login, Password auth) audit qilindi. |
| **10** | **Nginx** | Veb-server (HTTP daemon) | Ichki me’yoriy hujjatlar portali aktivi sifatida xizmat qildi; ma’lumotlar sizishi va shifrlanmagan aloqa zaifligini ko‘rsatish uchun foydalanildi. |
| **11** | **UFW (Uncomplicated Firewall)** | Xavfsizlik devori interfeysi | Linux yadrosining paket filtrlash tizimi (iptables/nftables) holatini audit qilish (`ufw status verbose`). |
| **12** | **systemd / journalctl** | Tizim va xizmatlar menejeri | Xizmatlarni avtomatik yuklanishga qo‘yish (`systemctl enable --now`), ularning faolligini tekshirish (`is-active`) va hodisalar jurnallarini tahlil qilish (`journalctl`). |
| **13** | **dpkg / apt vositalari** | Paketlar boshqaruvi | Dasturlar o‘rnatish, paket versiyalarini aniqlash (`dpkg-query -W`) va yangilanmagan paketlar ro‘yxatini olish (`apt list --upgradable`). |

---

## 3* Bosqichma-bosqich bajarilgan CLI buyruqlar, ularning argumentlari va qo‘llanish maqsadi

Mashg‘ulot davomida bajarilgan har bir buyruq ma’lum bir xavfsizlik yoki tizim sozlash maqsadiga xizmat qildi. Quyida ularning qadamma-qadam tahlili keltirilgan:

### 0-Bosqich: Host tizimida muhitni tayyorlash va VMlarni yaratish
* **1. Paket indekslarini yangilash va arxivator o‘rnatish:**
  ```bash
  sudo apt update
  sudo apt install -y p7zip-full
  ```
  * `sudo`: Administrator huquqi bilan bajarish.
  * `apt update`: Dasturlar ombori (repository) indekslarini yangilash.
  * `-y`: O‘rnatish jarayonidagi barcha savollarga avtomatik rozilik bildirish.
  * `p7zip-full`: `.7z` arxivlarini ochish uchun to‘liq utilita.
* **2. VMlar uchun alohida kataloglar yaratish:**
  ```bash
  mkdir -p ~/SEC-LAB/extracted ~/SEC-LAB/SEC-ANALYST ~/SEC-LAB/SRV-1
  ```
  * `-p` (*parents*): Ota kataloglar bo‘lmasa ularni ham bir vaqtda yaratish va xatolik chiqarmaslik.
* **3. Obrazni ochish va ikkita alohida disk nusxasini yaratish:**
  ```bash
  7z x UbuntuServer_24.04_VM.7z -o~/SEC-LAB/extracted/
  cp -r ~/SEC-LAB/extracted/* ~/SEC-LAB/SEC-ANALYST/
  cp -r ~/SEC-LAB/extracted/* ~/SEC-LAB/SRV-1/
  ```
  * `7z x`: Arxiv ichidagi fayllar daraxtini to‘liq o‘z holicha ochish.
  * `-o[yo'l]`: Natija tushadigan katalogni ko‘rsatish.
  * `cp -r`: Barcha fayllarni rekursiv nusxalash.
  * **Maqsadi:** Har bir virtual mashina o‘zining mustaqil diskiga (`.vmdk`) ega bo‘lishi shart. Bitta diskni ikki VMga biriktirish fayllar tizimini buzadi (disk corruption).
* **4. Host nomlarini berish (Mehmon VM konsolida):**
  ```bash
  sudo hostnamectl set-hostname SEC-ANALYST   # SEC-ANALYST da
  sudo hostnamectl set-hostname SRV-1         # SRV-1 da
  ```
  * **Maqsadi:** Tizimga doimiy nom berish, tarmoqda ularni adashtirib yubormaslik.

---

### 1-Bosqich: Izolyatsiyalangan tarmoqlarni sozlash
* **VMware Network Editor orqali:**
  * `vmnet8` (NAT) — vaqtincha internet berish uchun (DHCP yoqilgan).
  * `vmnet2` (Host-only) — `192.168.56.0/24`, **DHCP qat’iyan o‘chirilgan**.
  * **Maqsadi:** Laboratoriya tarmog‘ida begonalar bo‘lmasligi, avtomatik IP tarqatilmasligi va tarmoq tashqariga chiqmasligi.

---

### 2-Bosqich: Paketlar va xizmatlarni o‘rnatish (Internet mavjud paytda)
* **1. Begona proksi sozlamalarini tozalash (Shaxsiy Wi-Fi uchun):**
  ```bash
  unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY no_proxy NO_PROXY
  sudo rm -f /etc/apt/apt.conf.d/99proxy
  ```
  * *(Universitet muhitida esa aksincha: `export http_proxy="http://172.18.1.1:2500"` va `sudo tee /etc/apt/apt.conf.d/99proxy` orqali proksi kiritilgan).*
* **2. Internet aloqasini tekshirish:**
  ```bash
  ping -c 3 8.8.8.8
  ```
  * `-c 3`: Cheksiz ping yubormasdan, aynan 3 ta paketdan so‘ng to‘xtatish.
* **3. SEC-ANALYST da audit vositalarini o‘rnatish:**
  ```bash
  sudo apt install -y nmap curl iproute2 iputils-ping
  nmap --version | head -n 1
  ```
  * `nmap`: Tarmoq skaneri.
  * `head -n 1`: Versiyaning faqat birinchi satrini ko‘rsatish.
* **4. SRV-1 da server xizmatlarini o‘rnatish va faollashtirish:**
  ```bash
  sudo apt install -y openssh-server nginx ufw curl iproute2 iputils-ping
  sudo systemctl enable --now ssh nginx
  systemctl is-active ssh nginx
  ```
  * `enable --now`: Xizmatlarni ham avto-yuklanishga qo‘yish (`enable`), ham zudlik bilan ishga tushirish (`--now`).
  * `is-active`: Ikkala xizmat `active` holatdaligini tasdiqlash.
* **5. Lokal veb-xizmatni tekshirish:**
  ```bash
  curl -I http://127.0.0.1
  ```
  * `-I` (*HEAD so‘rov*): Sahifa kodini emas, faqat server HTTP sarlavhalarini tekshirish (`HTTP/1.1 200 OK`).

---

### 3-Bosqich: NAT adapterini uzish va A1-base snapshotini yaratish
* **Harakat:** VMware sozlamalarida Adapter 1 (NAT) dan "Connected" va "Connect at power on" belgilarini olib tashlash.
* **Buyruq:**
  ```bash
  sudo poweroff
  ```
* **VMware menyusida:** `VM -> Snapshot -> Take Snapshot` $\rightarrow$ Nomi: `A1-base`.
* **Maqsadi:** NAT uzilgach tizim to‘liq izolyatsiyalanadi. Snapshot esa kelgusida mashqlarni boshidan toza nuqtadan qayta boshlash uchun xavfsizlik kafolatidir.

---

### 4-Bosqich: SEC-LAB tarmog‘ida statik IP konfiguratsiyasi
* **Interfeys nomini aniqlash:** `ip -br link` (masalan, `ens37`).
* **SEC-ANALYST da IP biriktirish:**
  ```bash
  LAB_IF=ens37
  sudo ip link set "$LAB_IF" up
  sudo ip addr flush dev "$LAB_IF"
  sudo ip addr add 192.168.56.10/24 dev "$LAB_IF"
  ip -br addr show "$LAB_IF"
  ```
* **SRV-1 da IP biriktirish:**
  ```bash
  LAB_IF=ens37
  sudo ip link set "$LAB_IF" up
  sudo ip addr flush dev "$LAB_IF"
  sudo ip addr add 192.168.56.20/24 dev "$LAB_IF"
  ip -br addr show "$LAB_IF"
  ```
  * `link set dev up`: Tarmoq interfeysini yoqish.
  * `addr flush dev`: Eski, qolib ketgan noto‘g‘ri IP larni tozalash (konfliktlarning oldini olish).
  * `addr add .../24`: `/24` maskasi bilan statik manzil berish.
* **O‘zaro aloqani tekshirish (SEC-ANALYST dan):**
  ```bash
  ping -c 3 192.168.56.20
  ip route get 192.168.56.20
  curl -I --max-time 5 http://192.168.56.20
  ```
  * `ip route get`: Paket serverga aynan `ens37` orqali to‘g‘ridan-to‘g‘ri ketayotganini tekshirish.
  * `--max-time 5`: Agar javob bo‘lmasa, 5 soniyada to‘xtatish.

---

### 5-Bosqich: SRV-1 da lokal aktivlar inventarizatsiyasi
SRV-1 ichida aktivlarni aniqlash buyruqlari:
```bash
hostnamectl
ip -br addr
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS
free -h
sudo ss -lntup
systemctl --type=service --state=running --no-pager
dpkg-query -W openssh-server nginx ufw
```
* **Maqsadi va parametrlari:**
  * `hostnamectl`: OS versiyasi, arxitektura va Linux yadrosini ko‘rish (A-01 aktivi).
  * `lsblk`: Disk bo‘limlari va veb-fayllar turgan joyni aniqlash.
  * `free -h`: Tizim resurslarining yetarliligini baholash.
  * `sudo ss -lntup`:
    * `-l` (*listening*): Tinglayotgan soketlar.
    * `-n` (*numeric*): Portlarni raqamda ko‘rsatish (masalan, `80`).
    * `-t` (*tcp*), `-u` (*udp*): TCP va UDP protokollari.
    * `-p` (*processes*): Qaysi dastur va qaysi PID portni egallab turganini aniqlash.
  * `dpkg-query -W`: Paketlarning o‘rnatilgan aniq versiyalarini tekshirish.

---

### 6-Bosqich: Masofaviy skanerlash va audit (SEC-ANALYST dan)
```bash
nmap -sn 192.168.56.0/24
nmap -sT -sV --version-light -p 1-1024 192.168.56.20
curl -I --max-time 5 http://192.168.56.20
```
* **Maqsadi va parametrlari:**
  * `nmap -sn` (**E1 dalili**): Ping sweep (Host discovery). Portlarni skanerlamasdan, tarmoqdagi barcha faol qurilmalarni topadi (`.10`, `.20`, `.1`).
  * `nmap -sT` (**E2 dalili**): TCP Connect scan. To‘liq 3 tomonlama qo‘l berishish (SYN-SYN/ACK-ACK) orqali port ochiqligini aniqlaydi. Oddiy foydalanuvchida ham ishlaydigan standart usul.
  * `-sV --version-light`: Servis versiyalarini (OpenSSH 9.6p1, nginx 1.24.0) yengil zondlar yordamida aniqlash.
  * `-p 1-1024`: Standart (Well-Known) portlarni tekshirish.
* **Nmap va `ss` solishtiruvi:**
  * `Nmap` — bu tashqi hujumchining nigohi (Tashqi hujum yuzasi / Attack surface).
  * `ss` — serverning ichki haqiqati (Yadro darajasida kim tinglamoqda).
  * Agar `ss` da port ochiq bo‘lsa-yu, Nmap da `filtered` yoki `closed` chiqsa — demak oraliqda xavfsizlik devori (Firewall) bor.

---

### 7-Bosqich: Himoya nazoratlari auditi va E1–E8 dalillarini yig‘ish (SRV-1 da)
```bash
sudo ufw status verbose
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication|maxauthtries)'
sudo ss -lntup
curl -I --max-time 5 http://192.168.56.20
apt list --upgradable 2>/dev/null
sudo journalctl -u ssh -n 20 --no-pager
sudo journalctl -u nginx -n 20 --no-pager
```
* **Maqsadi va parametrlari:**
  * `sudo ufw status verbose` (**E4 dalili**): Firewall o‘chiqligini (`Status: inactive`) fosh qilish.
  * `sshd -T | grep -E` (**E5 dalili**): Amaldagi xotiradagi SSH konfiguratsiyasini tekshirish. `permitrootlogin yes` va `passwordauthentication yes` ekanligi — eng jiddiy zaiflik.
  * `curl -I` (**E6 dalili**): `Server: nginx/1.24.0 (Ubuntu)` javobi orqali xizmat versiyasi oshkor bo‘layotganini (Information Disclosure) ko‘rsatish.
  * `apt list --upgradable` (**E7 dalili**): Yangilanish kutayotgan paketlarni aniqlash.
  * `journalctl -u ssh -n 20 --no-pager` (**E8 dalili**): Tizimga kirish loglarini tekshirish (Audit izi).

---

## 4* Self-study yo‘l xaritasi (Roadmap): Qaysi darajaga yetganda bularni o‘rganish kerak?

Ushbu laboratoriya ishi Axborot xavfsizligi va Tarmoq muhandisligi sohasida **Fundamental tayanch poydevor (Level 1 / Junior)** hisoblanadi. O‘z ustingizda mustaqil ishlayotganingizda bilim darajangizga qarab quyidagi bosqichma-bosqich o‘sish yo‘l xaritasiga amal qilishingiz tavsiya etiladi:

```
+-------------------------------------------------------------------------+
|                  KIBERXAVFSIZLIK VA TARMOQ ROADMAP                      |
+-------------------------------------------------------------------------+
                                    |
   [Boshlang'ich (Foundation)]      v
   - Linux CLI, Bash asoslari
   - Tarmoq: OSI modeli, TCP/IP, Subnetting (/24), DNS, ARP
   - Virtualizatsiya: VMware, Host-Only, NAT, Bridged
                                    |
   [Junior (L1) SOC / Security Eng] v   <--- (HOZIRGI DARANGIZ: A1 LAB)
   - Port skanerlash: Nmap (TCP, UDP, Scripting Engine - NSE)
   - Traffic sniffing: Wireshark, tcpdump
   - Linux Hardening: UFW/iptables, SSH kalitlari, fail2ban
   - Log tahlili: Syslog, journalctl, auth.log
   - Risk metodologiyasi: CIA triadasi, R = E x T, NIST SP 800-30
                                    |
   [Middle (L2) Penetration Tester] v
   - Zaifliklarni skanerlash: OpenVAS, Nessus
   - Ekspluatatsiya: Metasploit, Hydra (Brute-force), Burp Suite
   - Himoya tizimlari: Wazuh SIEM, Suricata IDS/IPS
   - Avtomatlashtirish: Python / Bash skriptlar bilan audit
                                    |
   [Senior / Security Architect]    v
   - Zero Trust arxitekturasi, Tarmoq mikrosegmentatsiyasi
   - Threat Modeling: STRIDE, MITRE ATT&CK matritsasi
   - ISO/IEC 27001, SOC2, CIS Benchmarks standartlari
```

### 1-bosqich: Hozirgi daraja (Foundation $\rightarrow$ Junior L1 darajasi)
* **Qachon o‘rganiladi:** Universitetning 3-kursida yoki IT/Security yo‘nalishini noldan boshlagandan 2–4 oy o‘tgach.
* **Nimalar o‘zlashtirilishi shart:**
  * Linux buyruqlar satrida erkin ishlash (`systemctl`, `journalctl`, `ip`, `ss`, `grep`, `chmod`, `chown`).
  * Tarmoq paketlarining harakati: L2 (MAC) $\rightarrow$ L3 (IP) $\rightarrow$ L4 (TCP/UDP portlar) $\rightarrow$ L7 (HTTP/SSH protokollari).
  * Nmap buyruqlari va ularning ma’nosi (Ping sweep, TCP Connect, SYN scan, Version detection).
  * Risk tushunchasi: Aktiv, Tahdid, Zaiflik, Nazorat va Risk o‘rtasidagi nozik farqlar.

### 2-bosqich: Junior $\rightarrow$ Pre-Middle darajasi (Keyingi 3–6 oy)
* **O‘rganiladigan mavzular:**
  * **Amaliy hujum va himoya:** Nmap NSE skriptlari (`--script vuln`), parollarni sinovdan o‘tkazish (Hydra bilan SSH ga brute-force qilib, loglarda qanday aks etishini ko‘rish).
  * **Trafikni tahlil qilish:** `tcpdump` va `Wireshark` vositasida shifrlanmagan HTTP so‘rovlarini tutib olish (sniffing) va HTTPS ning afzalligini amalda ko‘rish.
  * **Tizimni mustahkamlash (Hardening):** Faqat tavsiya yozib qolmay, UFW qoidalarini amalda yozish, SSH ni faqat Ed25519 kalitli qilish, `fail2ban` o‘rnatib, noto‘g‘ri kirishlarni avtomatik bloklash.

### 3-bosqich: Middle darajasi (6–18 oy)
* **O‘rganiladigan mavzular:**
  * **SIEM va Monitoring:** Wazuh yoki Elastic Security o‘rnatib, server loglarini markazlashtirilgan holda tahlil qilish.
  * **Xavfsizlik skanerlari:** Nessus, OpenVAS (Greenbone) orqali avtomatlashgan zaifliklar tahlili.
  * **Veb-xavfsizlik:** OWASP Top-10 zaifliklari, Burp Suite bilan ishlash.
  * **MITRE ATT&CK:** Hujumchilarning xatti-harakatlarini global taktikalar va texnikalar bilan bog‘lash.

### 4-bosqich: Senior / Architect darajasi (2+ yil)
* **O‘rganiladigan mavzular:**
  * Korporativ xavfsizlik siyosati (ISO 27001, NIST CSF).
  * Mikrosegmentatsiya va Zero Trust tamoyillari.
  * Incident Response (Hodisalarga javob berish) va Forensic tahlil.

---

## 5* O‘zimning siz uchun qo‘shimcha maslahatlarim

Xavfsizlik sohasida tez va mustahkam rivojlanishingiz uchun amaliy tavsiyalar:

1. **"Buyruqlarni yodlamang — Mantiqni tushuning":**
   * Masalan, `nmap -sT` nima uchun sekinroq, `nmap -sS` esa nega stealth (yashirin) deb ataladi? Chunki `-sT` to‘liq 3-bosqichli TCP handshake (SYN $\rightarrow$ SYN/ACK $\rightarrow$ ACK) qiladi va loglarda iz qoldiradi. `-sS` esa oxirgi ACK o‘rniga RST yuboradi. Buni Wiresharkda ko‘rsangiz, umrbod esingizda qoladi.

2. **"Zaiflik $\ne$ Risk" ekanligini doimo farqlang:**
   * Ko‘pchilik boshlovchilar ochiq 22 yoki 80-portni ko‘rib, "Bu risk!" deb yozishadi. Bu xato! Ochiq port — bu bor-yo‘g‘i tizimning ishlash funksiyasi. Agar o‘sha portdagi dasturda zaiflik bo‘lsa (masalan, root paroli oson bo‘lsa) va tarmoqda unga hujum qila oladigan tajovuzkor mavjud bo‘lsa — mana shundagina u **Risk** hisoblanadi.

3. **Virtual laboratoriyalarda "Hardening"ni o‘zingiz amalda yakunlang:**
   * A1 laboratoriyasida siz zaifliklarni aniqladingiz va jadvalda tavsiya yozdingiz. Endi o‘zingiz mustaqil ravishda:
     * `sudo ufw default deny incoming` va `sudo ufw allow 22/tcp` buyruqlarini bering.
     * `ssh-keygen` orqali kalit yaratib, parolli kirishni o‘chiring (`PasswordAuthentication no`).
     * So‘ng SEC-ANALYST dan yana Nmap va curl qilib ko‘ring. Dalillar qanday o‘zgarganini va risk balli qanday pasayganini o‘z ko‘zingiz bilan ko‘ring. Bu sizga 100% amaliy tajriba beradi!

4. **Doimiy "Snapshot" madaniyatiga odatlaning:**
   * Xavfsizlikda tajriba o‘tkazayotganda tizimni buzib qo‘yish tabiiy hol. Har qanday jiddiy o‘zgarishdan oldin VMda Snapshot oling. Bu real serverlarda ishlaganda ham ehtiyotkorlik instinktini shakllantiradi.

5. **Tavsiya etiladigan amaliy platformalar:**
   * **OverTheWire (Bandit):** Linux CLI, ruxsatlar va xavfsizlikni o‘rganish uchun dunyodagi eng zo‘r bepul interaktiv o‘yin.
   * **TryHackMe:** "Pre-Security" va "Cyber Defense" yo‘llari (Nmap, Wireshark, Linux fundamental darslari judayam tushunarli).
   * **Hack The Box:** Starting Point mashqlari.

6. **O‘z Portfoliongizni (GitHub / Writeup) shakllantiring:**
   * Bajarayotgan har bir laboratoriya ishingiz (A1, A2...) bo‘yicha GitHub’da chiroyli `markdown` hisobotlar saqlab boring. Kelgusida xavfsizlik bo‘yicha ishga kirishda (intervyularda) bunday tahliliy hisobotlar sizning eng kuchli rezyumeingizga aylanadi.

---

## Xulosa

`A1` laboratoriya ishi — bu tarmoq xavfsizligi mutaxassisining "ko‘zi" va "mantiqiy apparati"ni shakllantiruvchi darsdir. Unda talaba Nmap va `ss` vositalari orqali tizimga tashqaridan va ichkaridan qarashni, dalillarni tizimlashtirishni, UFW va SSH konfiguratsiyasidagi kamchiliklarni fosh qilishni hamda eng asosiysi — xavflarni matematik-mantiqiy baholashni o‘rganadi. Ushbu poydevor keyingi laboratoriyalarda (firewall sozlash, hujumlarni qaytarish va monitoring tizimlarini qurishda) eng asosiy tayanch bo‘lib xizmat qiladi.
