# TARMOQ XAVFSIZLIGI | A11: SSH VA RDP XIZMATLARINI XAVFSIZ SOZLASH (HARDENING REMOTE ACCESS)
## 1-Variant: Masofaviy Kirishni Cheklash, Autentifikatsiya va Tekshirish Bo‘yicha To‘liq Amaliy Qo‘llanma

---

## 📌 ISHCHI MUHIT VA FOYDALANUVCHI SHARTLARI (LOKAL WI-FI VA UBUNTU HOST + VMWARE)

Ushbu amaliy qo‘llanma foydalanuvchining real ishchi muhitiga to‘liq moslashtirilgan holda tayyorlangan:
1. **Tarmoq Muhiti — Lokal/Uy Wi-Fi:**
   - Siz universitet Wi-Fi tarmog‘idan emas, **o‘zingizning shaxsiy lokal Wi-Fi tarmog‘ingizdan** foydalanasiz.
   - Barcha laboratoriya sinovlari va amaliyotlar tashqi lokal Wi-Fi dan to‘liq izolyatsiya qilingan **VMware Host-Only (`vmnet1`)** tarmog‘ida (`10.110.10.0/24`) amalga oshiriladi.
   - Bu sizning real uyingizdagi Wi-Fi trafigiga, internet tezligiga yoki routeringizga umuman ta’sir qilmaydi, tashqi tarmoqdan kiberxavflarni nolga tushiradi va tashqi IP to‘qnashuvlarning oldini oladi.

2. **Host Operatsion Tizimi — Ubuntu Linux (Windows Host EMAS!):**
   - Sizning asosiy kompyuteringiz — **Ubuntu Linux** Desktop.
   - Virtualizatsiya dasturi: Ubuntu ichiga o‘rnatilgan **VMware Workstation Pro / Player**.
   - Barcha boshqaruv buyruqlari, SSH kalitlari yaratish, serverlarga kirish va RDP mijoz dasturlari (`xfreerdp`, `remmina`, `ssh`, `netcat`) to‘g‘ridan-to‘g‘ri **Ubuntu terminali** orqali bajariladi.

3. **Virtual Mashinalar (VMware ichidagi topologiya):**
   - **Boshqaruvchi Host (ADMIN):** Sizning asosiy Ubuntu tizimingiz — `10.110.10.1/24` (Yagona ruxsat berilgan boshqaruv nuqtasi).
   - **SSH-SRV:** Ubuntu Server 24.04 LTS VM — `10.110.10.10/24` (OpenSSH server, UFW xavfsizlik devori, `labssh` cheklangan hisobi).
   - **RDP-SRV:** VMware ichida oldindan sozlab qo‘yilgan **Windows 11 Desktop VM** — `10.110.10.20/24` (Masofaviy ish stoli (RDP) xizmati, NLA, TLS, Clipboard va Disk uzatishni taqiqlash siyosatlari, Windows Defender Firewall, `labrdp` va `labdeny` hisoblari).
   - **TEST-VM:** Ubuntu Server 24.04 LTS VM — `10.110.10.30/24` (Ruxsatsiz begona manba; Netcat yordamida firewall port blokirovkasini isbotlash uchun ishlatiladi).

---

## 1. MAVZUNING ASOSIY MAQSADI VA XAVFSIZLIK FALSAFASI

### 1.1. Laboratoriya ishining maqsadi
Mazkur amaliy mashg‘ulotning asosiy maqsadi — tashkilot va korxonalarda serverlarni masofadan boshqarish uchun keng qo‘llaniladigan **SSH (Secure Shell - 22/TCP)** va **RDP (Remote Desktop Protocol - 3389/TCP)** protokollarini ishlab chiqarish (production) darajasida xavfsiz sozlash (**hardening**), ularning standart zaifliklarini bartaraf etish hamda amaliy sinovlar orqali ushbu himoya tizimlarining ishlashini to‘liq isbotlashdan iborat.

### 1.2. Amaliyotda joriy qilinadigan 5 ta asosiy xavfsizlik ustuni:
1. **Server shaxsini tekshirish va yaxlitlik (Server Identity & Integrity):**
   - Serverga ulanishdan oldin uning kriptografik pasporti — **SSH Host Key (Ed25519)** va **RDP TLS sertifikati** barmoq izlarini (fingerprint/thumbprint) server konsolidagi haqiqiy qiymatlar bilan solishtirish.
   - Bu orqali tarmoqda oraliq hujumchi soxta server qo‘yib ma’lumotlarni o‘g‘irlashi — **Man-in-the-Middle (MITM)** hujumlarining oldi olinadi.
2. **Minimal imtiyoz tamoyili (Principle of Least Privilege):**
   - Masofadan tizimga to‘g‘ridan-to‘g‘ri eng yuqori imtiyozli hisoblar (`root` yoki Windows `Administrator`) bilan kirishni to‘liq taqiqlash.
   - Masofaviy kirish huquqini faqat alohida oddiy hisoblarga (`labssh`, `labrdp`) berish.
   - Huquqi yo‘q foydalanuvchilar (`labdeny`) uchun to‘g‘ri parol bo‘lsa ham tizimga kirishni avtorizatsiya darajasida rad etish.
3. **Mustahkam kriptografik autentifikatsiya (Strong Cryptographic Authentication):**
   - Lug‘at va parollarni taxmin qilish (brute-force) hujumlariga ochiq bo‘lgan oddiy parollardan voz kechish.
   - SSH uchun faqat zamonaviy assimetrik **Ed25519** kalit juftligini majburiy qilish (`AuthenticationMethods publickey`).
   - RDP uchun foydalanuvchini to‘liq grafik ish stoli ochilishidan oldin tarmoq darajasida tekshiruvchi **NLA (Network Level Authentication)** mexanizmini va **TLS** xavfsizlik qatlamini majburiy yoqish.
4. **Tarmoq perimetrini qat’iy chegaralash (Network Access Control - Firewall):**
   - Xizmat portlarini butun tarmoqqa ochiq qoldirmaslik.
   - Xavfsizlik devorlari (**Linux UFW** va **Windows Defender Firewall**) orqali **"Default Deny Incoming"** (hamma kiruvchi oqimni taqiqlash) tamoyilini joriy qilish.
   - Faqat yagona tasdiqlangan boshqaruv IP-manzili (`10.110.10.1` - Ubuntu Host) dan 22 va 3389 portlariga kirishga ruxsat berish, begona manbalarni (shu jumladan `10.110.10.30` TEST-VM) so‘zsiz bloklash.
5. **Ma’lumotlar sizib chiqishini to‘sish (Data Exfiltration Prevention):**
   - Masofaviy RDP sessiyasi orqali bufer almashinuvi (**Clipboard Redirection**) va lokal qattiq disklarni serverga ulashni (**Drive Redirection**) Windows guruh siyosatlari darajasida taqiqlash.
   - Bu orqali xodim serverdagi maxfiy fayllarni o‘z kompyuteriga olib chiqib keta olmasligi va xodim kompyuteridagi ransomware/virus serverga tarqala olmasligi ta’minlanadi.

---

## 2. HAR BIR MAVZU BO‘YICHA TO‘LIQ QO‘LLANMA VA NAZARIY ASOSLAR

### 2.1. Tarmoq Topologiyasi va IP Manzillash Arxitekturasi

Laboratoriya to‘liq VMware Workstation ichidagi **Host-Only (`vmnet1`)** tarmog‘ida quriladi:

| Tugun Nomi | Muhit / Operatsion Tizim | IP Manzil / Maska | Ochiq Xizmat / Port | Vazifasi va Roli |
| :--- | :--- | :--- | :--- | :--- |
| **Ubuntu Host (ADMIN)** | Jismoniy kompyuter (Ubuntu Desktop) | `10.110.10.1/24` | Mijoz: OpenSSH, `xfreerdp`, `nc` | Administrator boshqaruv nuqtasi. **Yagona ruxsatli manba!** |
| **SSH-SRV** | VMware VM (Ubuntu Server 24.04 LTS) | `10.110.10.10/24` | 22/TCP (OpenSSH) | SSH server, UFW xavfsizlik devori, `labssh` hisobi |
| **RDP-SRV** | VMware VM (Windows 11 Desktop) | `10.110.10.20/24` | 3389/TCP (RDP) | RDP server, NLA, TLS, Clipboard/Drive cheklovi, `labrdp`/`labdeny` |
| **TEST-VM** | VMware VM (Ubuntu Server 24.04 LTS) | `10.110.10.30/24` | Mijoz: `netcat-openbsd` | Ruxsatsiz begona manba. Firewall blokirovkasini isbotlash vositasi |

```
                 [ SIZNING SHAXSIY LOKAL WI-FI TARMOg‘INGIZ ]
                                     │ (To‘liq mustaqil, Labga ta’siri yo‘q)
                                     ▼
                   ╔═════════════════════════════════════╗
                   ║   UBUNTU HOST (Sizning tizimingiz)  ║
                   ║   Interfeys: vmnet1                 ║
                   ║   IP: 10.110.10.1/24                ║
                   ║   Roli: Yagona Ruxsatli Boshqaruvchi║
                   ╚═════════════════════════════════════╝
                                     │
   ══════════════════════════════════╪══════════════════════════════════ (VMware Host-Only: 10.110.10.0/24)
                  │                  │                  │
                  ▼                  ▼                  ▼
         ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
         │    SSH-SRV     │ │    RDP-SRV     │ │    TEST-VM     │
         │ Ubuntu Server  │ │ Windows 11 VM  │ │ Ubuntu Server  │
         │ 10.110.10.10   │ │ 10.110.10.20   │ │ 10.110.10.30   │
         │ Port: 22/TCP   │ │ Port: 3389/TCP │ │ Ruxsatsiz manba│
         └────────────────┘ └────────────────┘ └────────────────┘
```

### 2.2. VMware Workstation (Ubuntu) da Host-Only tarmoqni to‘g‘ri sozlash:
1. Ubuntu tizimingizda VMware Workstation dasturini oching.
2. Menyudan **Edit** → **Virtual Network Editor** ni bosing (yoki terminalda `sudo vmware-netcfg` buyrug‘ini bering).
3. **vmnet1** adapterini tanlang va quyidagi parametrlarni tekshiring:
   - **Type:** Host-only.
   - **Subnet IP:** `10.110.10.0`
   - **Subnet mask:** `255.255.255.0`
   - **Use local DHCP service to distribute IP address to VMs:** Belgini **OLIB TASHLANG** (DHCP o‘chiq bo‘lishi shart, barcha IP lar qo‘lda statik beriladi!).
   - **Connect a host virtual adapter to this network:** Belgini **YOQING** (bu sizning Ubuntu hostingizga `vmnet1` virtual interfeysini taqdim etadi).
4. Virtual mashinalarning (SSH-SRV, RDP-SRV, TEST-VM) har birining sozlamalarida (`Settings` → `Network Adapter`) **Custom: Specific virtual network** bandidan aynan **VMnet1 (Host-only)** ni tanlang.
5. Mashinalarda NAT yoki Bridged adapterlar bo‘lmasin (ularni uzib qo‘ying).

---

### 2.3. SSH Kriptografik Arxitekturasi: Host Key vs User Key
Ko‘pchilik yangi boshlovchilar ikkita muhim kalit turini adashtirishadi:
- **Server Host Key (`/etc/ssh/ssh_host_*`):**
  - Bu serverning shaxsiy kriptografik guvohnomasi (pasporti).
  - OpenSSH server o‘rnatilganda avtomatik generatsiya qilinadi.
  - Server mijozga o‘zining haqiqiyligini isbotlash uchun ishlatiladi ("Men aynan siz qidirgan 10.110.10.10 serveriman, oraliqda hech kim soxta server qo‘ygani yo‘q").
  - Mijoz serverga birinchi bor ulanganda ushbu kalitning SHA256 barmoq izi (fingerprint) mijozning `~/.ssh/known_hosts` fayliga saqlanadi.
- **Foydalanuvchi Kaliti (User Key — `a11_ed25519` va `a11_ed25519.pub`):**
  - Bu mijozning (sizning) shaxsingizni tasdiqlovchi kalit juftligi.
  - **Shaxsiy kalit (Private Key):** Faqat sizning Ubuntu hostingizda saqlanadi va maxfiy tutiladi.
  - **Ochiq kalit (Public Key):** Serverdagi `~/.ssh/authorized_keys` fayliga ko‘chirib qo‘yiladi.
  - Server sizga matematik jumboq yuboradi, sizning shaxsiy kalitingiz uni yechadi va server hech qanday parol uzatmasdan sizni tizimga kiritadi.
- **Passphrase vs Foydalanuvchi Paroli:**
  - `Passphrase` — bu shaxsiy kalit faylini diskda AES bilan shifrlab turuvchi maxfiy so‘z. U faqat sizning **Ubuntu hostingizda** lokal tekshiriladi, hech qachon tarmoqqa chiqmaydi.
  - Foydalanuvchi paroli esa serverdagi `/etc/shadow` faylida saqlanadi va tarmoq orqali uzatiladi (bruteforce ga ochiq). Kalitli autentifikatsiya yoqilganda bu parol butunlay chetlab o‘tiladi.

---

### 2.4. Nega Ed25519 Algoritmi?
Laboratoriyada eskirgan RSA (2048/4096 bit) o‘rniga zamonaviy **Ed25519** (Edwards-curve Digital Signature Algorithm — Curve25519) algoritmidan foydalaniladi:
- Hajmi ixcham: kalit uzunligi bor-yo‘g‘i 256-bit, ochiq kalit 68 belgidan iborat.
- RSA 3072-bit xavfsizlik darajasidan yuqori kriptografik chidamlilikka ega.
- Doimiy vaqtli (constant-time) algoritmlash tufayli yon kanal (side-channel) va kesh tahlili hujumlariga mutlaqo chidamli.
- Yuqori tezlikda kalit hosil qiladi va imzolaydi.

---

### 2.5. RDP Arxitekturasi: Standart RDP vs NLA va TLS
- **Standart (Legacy) RDP:**
  - Mijoz 3389 portga ulanishi bilanoq, server to‘liq Windows grafik sessiyasini boshlaydi, xotiradan katta resurs ajratadi, login ekranini chizadi va mijozga uzatadi.
  - Bu serverni **Denial-of-Service (DoS)** hujumlariga juda zaif qilib qo‘yadi (hujumchi minglab soxta ulanishlar bilan Windows xotirasini to‘ldirishi mumkin).
- **Network Level Authentication (NLA) — CredSSP orqali:**
  - Foydalanuvchi ma’lumotlari to‘liq Windows grafik oynasi ochilishidan **OLDIN**, tarmoq darajasida TLS shifrlangan kanal orqali tekshiriladi.
  - Faqat to‘g‘ri login va paroli bor foydalanuvchiga grafik interfeys ajratiladi. Bu server protsessori va xotirasini behuda sarflanishdan asraydi.
- **TLS (SecurityLayer = 2):**
  - RDP ning eski o‘ziga xos shifrlashidan voz kechib, standart TLS (Transport Layer Security) protokolidan foydalanadi. Ma’lumotlar yo‘lda tinglanishi (eavesdropping) va o‘zgartirilishi mumkin emas.

---

### 2.6. RDP Clipboard va Drive Redirection Xavfi
RDP orqali serverga ulanganda standart holda bufer (nusxa olish/qo‘yish) va lokal qattiq disklar avtomatik ulanadi:
- **Xavf 1 (Ma’lumot o‘g‘irlanishi — Data Exfiltration):** Ichki xodim serverdagi maxfiy mijozlar bazasi yoki ma’lumotlarni nusxalab o‘z shaxsiy kompyuteriga olib chiqishi mumkin.
- **Xavf 2 (Zararli dasturlar kirib kelishi):** Xodimning kompyuterida virus yoki ransomware bo‘lsa, u ulangan virtual disklar orqali korporativ serverga yuqishi mumkin.
- Shu sababli Windows Registry va Guruh Siyosatlarida `fDisableClip = 1` va `fDisableCdm = 1` qilib bu kanallar server tomonida qat’iyan to‘sib qo‘yiladi.

---

### 2.7. Xavfsizlik Devorida "Tor Ruxsat" vs "Keng Ruxsat" Ziddiyati
Xavfsizlik devorlari (iptables/nftables, UFW, Windows Firewall) qoidalarni birinchi mos kelish tamoyili (first match) bo‘yicha tekshiradi:
- Agar tizimda umumiy `ALLOW ANY from ANY to 22/tcp` (keng ruxsat) qoidasi mavjud bo‘lsa, siz uning yoniga `ALLOW from 10.110.10.1 to 22/tcp` (tor ruxsat) qo‘shganingiz bilan server himoyalanib qolmaydi! Begona kompyuter paket yuborganda, u tor qoidadan o‘tmasa ham, keng qoidaga tushib ruxsat oladi.
- Shu sababli har doim:
  1. Barcha eski keng ruxsat qoidalari o‘chiriladi.
  2. Sukut bo‘yicha kiruvchi barcha paketlar taqiqlanadi (**Default Inbound: Block / Deny**).
  3. Faqat aniq boshqaruv IP manziliga (`10.110.10.1`) tor qoida qo‘shiladi.

---

## 3. MASHQNI BAJARISH BO‘YICHA KETMA-KET QO‘LLANMA VA HAR BIR CLI BUYRUQNING TO‘LIQ TAHLILI

Laboratoriya mashg‘ulotini boshlashdan oldin barcha virtual mashinalaringiz (SSH-SRV, RDP-SRV, TEST-VM) VMware da ishga tushirilganligiga ishonch hosil qiling.

---

### 1-BOSQICH: Tarmoq Manzillarini Sozlash va Tayyorgarlik

#### 1.1. Ubuntu Host (ADMIN) da IP manzilni sozlash
Sizning jismoniy Ubuntu kompyuteringizda VMware'ning `vmnet1` virtual interfeysiga `10.110.10.1` manzilini beramiz.

Ubuntu host terminalida:
```bash
sudo ip link set vmnet1 up
sudo ip addr replace 10.110.10.1/24 dev vmnet1
ip -br addr show vmnet1
```

🔍 **CLI buyruqlari va bayroqlari tahlili:**
- `sudo`: Buyruqni tizim ma’muri (root) imtiyozi bilan bajarish.
- `ip link set vmnet1 up`: Linux yadro darajasida `vmnet1` virtual tarmoq interfeysini faollashtiradi (apparat holatini "UP" qiladi).
- `ip addr replace 10.110.10.1/24 dev vmnet1`: `vmnet1` interfeysiga statik `10.110.10.1` IP manzilini va `/24` (255.255.255.0) tarmoq niqobini biriktiradi. `replace` bayrog‘i avval boshqa manzil bo‘lsa xatoliksiz yangisiga almashtiradi.
- `ip -br addr show vmnet1`: `-br` (brief — qisqa) formatda interfeys nomi, holati (UP) va uning IPv4 manzilini ko‘rsatadi.

Ubuntu hostga kerakli boshqaruv vositalarini o‘rnatamiz:
```bash
sudo apt update && sudo apt install -y openssh-client freerdp2-x11 netcat-openbsd
```

🔍 **Paketlar tahlili:**
- `apt update`: Repozitoriyalardagi paketlar katalogini yangilaydi.
- `apt install -y`: Paketlarni foydalanuvchidan tasdiq so‘ramasdan (`-y` = yes) avtomatik o‘rnatadi.
- `openssh-client`: SSH mijoz dasturlari to‘plami (`ssh`, `ssh-keygen`).
- `freerdp2-x11`: Linux muhitida RDP ulanishlarini amalga oshiruvchi kuchli buyruqlar satri vositasi (`xfreerdp`).
- `netcat-openbsd`: Tarmoq portlarini tekshirish uchun qulay `nc` dasturi.

---

#### 1.2. SSH-SRV (Ubuntu Server) konsolida IP sozlash
VMware oynasidan **SSH-SRV** virtual mashinasi konsoliga kiring va buyruqlarni bajaring:

```bash
sudo -i
export TZ=UTC
ip -br link
```
Interfeys nomini aniqlaymiz (masalan `ens33`, `ens160` yoki `enp0s8`):
```bash
IF=$(ip -br link | awk '$1 !~ /lo/ {print $1; exit}')
echo "Tanlangan interfeys: $IF"

ip link set "$IF" up
ip addr replace 10.110.10.10/24 dev "$IF"
mkdir -p /root/A11
cp -a /etc/ssh /root/A11/ssh-before
ip -br addr show "$IF"
```

🔍 **CLI buyruqlari tahlili:**
- `sudo -i`: To‘liq administrator (root) muhitiga o‘tish.
- `export TZ=UTC`: Vaqt zonasini UTC ga o‘rnatish (kelgusida loglar tahlilida vaqt chalkashmasligi uchun).
- `IF=$(ip -br link | awk '$1 !~ /lo/ {print $1; exit}')`: `awk` skripti orqali `lo` (loopback) bo‘lmagan birinchi haqiqiy tarmoq kartasi nomini olib, `IF` o‘zgaruvchisiga joylaydi.
- `ip link set "$IF" up`: Tarmoq kartasini faollashtiradi.
- `ip addr replace 10.110.10.10/24 dev "$IF"`: Serverga `10.110.10.10/24` statik manzilini biriktiradi.
- `mkdir -p /root/A11`: `-p` bayrog‘i ota kataloglar mavjud bo‘lmasa ham xatosiz yaratadi; laboratoriya konfiguratsiyalari uchun papka ochadi.
- `cp -a /etc/ssh /root/A11/ssh-before`: `-a` (archive — barcha fayl huquqlari, egalari va vaqtlarini saqlab nusxalash) kaliti bilan standart `/etc/ssh` katalogining to‘liq zaxira nusxasini oladi.

---

#### 1.3. TEST-VM (Ubuntu Server) konsolida IP sozlash
VMware oynasidan **TEST-VM** konsoliga kirib, unga `10.110.10.30` manzilini beramiz:

```bash
sudo -i
TEST_IF=$(ip -br link | awk '$1 !~ /lo/ {print $1; exit}')
ip link set "$TEST_IF" up
ip addr replace 10.110.10.30/24 dev "$TEST_IF"
ip -br addr show "$TEST_IF"
```

🔍 **CLI buyruqlari tahlili:**
- TEST-VM virtual mashinasiga `10.110.10.30/24` manzili o‘rnatiladi. U tajribada "ruxsatsiz begona foydalanuvchi/hujumchi" rolini bajaradi.

---

#### 1.4. RDP-SRV (Windows 11 Desktop VM) konsolida IP sozlash
VMware ichidagi Windows 11 mashinasiga kiring, **Start** tugmasini bosing, `PowerShell` deb yozing va o‘ng tugmani bosib **Run as Administrator** (Administrator sifatida ishga tushirish) ni tanlang:

```powershell
New-Item -ItemType Directory C:\A11 -Force
Get-NetAdapter | Format-Table Name,InterfaceDescription,MacAddress,Status
```
Adapter nomi aniqlangach (odatda `Ethernet0` yoki `Ethernet`):
```powershell
$labNic = (Get-NetAdapter | Where-Object Status -eq 'Up')[0].Name
New-NetIPAddress -InterfaceAlias $labNic -IPAddress 10.110.10.20 -PrefixLength 24
Get-NetIPAddress -InterfaceAlias $labNic -AddressFamily IPv4
hostname
```

🔍 **PowerShell buyruqlari tahlili:**
- `New-Item -ItemType Directory C:\A11 -Force`: `C:\` diskida `A11` papkasini majburiy ravishda (`-Force`) yaratadi.
- `Get-NetAdapter`: Kompyuterdagi barcha jismoniy va virtual tarmoq adapterlarini ko‘rsatadi.
- `$labNic = ...`: Holati "Up" (yoqilgan) bo‘lgan birinchi adapterning nomini o‘zgaruvchiga oladi.
- `New-NetIPAddress -InterfaceAlias $labNic -IPAddress 10.110.10.20 -PrefixLength 24`: Belgilangan adapterga statik `10.110.10.20` manzilini va 24-bitli niqobni (255.255.255.0) biriktiradi.
- `hostname`: Windows mashinasining haqiqiy kompyuter nomini ekranga chiqaradi (ushbu nom keyinchalik RDP orqali kirishda zarur bo‘ladi).

---

### 2-BOSQICH: SSH Cheklangan Foydalanuvchisi va Server Shaxsini (Host Key) Tekshirish

#### 2.1. SSH-SRV da alohida oddiy foydalanuvchi yaratish
SSH xizmatiga hech qachon `root` hisobi bilan kirmaslik kerak. Shu sababli oddiy, `sudo` guruhida bo‘lmagan `labssh` hisobini ochamiz.

**SSH-SRV konsolida:**
```bash
adduser labssh
id labssh
```
*(Buyruq bajarilganda parol so‘raydi, masalan kuchli parol kiriting: `LabPass123!`).*

Xizmat holatini tekshiramiz:
```bash
systemctl disable --now ssh.socket 2>/dev/null || true
systemctl enable ssh.service
systemctl restart ssh.service
ss -lntp 'sport = :22'
```

Serverning Ed25519 Host Key barmoq izini (Fingerprint) olamiz:
```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub -E sha256 | tee /root/A11/server_hostkey_fingerprint.txt
```

🔍 **CLI buyruqlari va bayroqlari tahlili:**
- `adduser labssh`: Yangi `labssh` hisobini, uning uy katalogini (`/home/labssh`) va asosiy guruhini yaratadi.
- `id labssh`: Foydalanuvchining UID, GID va guruhlarini ko‘rsatadi. Chiqqan ro‘yxatda `sudo` yoki `admin` guruhi yo‘qligiga ishonch hosil qilinadi.
- `systemctl disable --now ssh.socket`: Ubuntu 24.04 da SSH sukut bo‘yicha `systemd socket activation` orqali ishlashi mumkin. Standart xizmat holatiga o‘tish uchun socketni to‘xtatib (`--now`) va avto-yuklanishini o‘chiramiz (`disable`).
- `systemctl enable ssh.service && systemctl restart ssh.service`: SSH xizmatini an’anaviy tizim xizmati sifatida faollashtiradi va qayta ishga tushiradi.
- `ss -lntp 'sport = :22'`: Soketlar statistikasini chiqaradi:
  - `-l`: faqat tinglayotgan (listening) portlar.
  - `-n`: IP va portlarni raqamli ko‘rsatish.
  - `-t`: TCP soketlar.
  - `-p`: jarayon nomi va PID ni ko‘rsatish.
  - `'sport = :22'`: manba porti 22 ga teng bo‘lgan qatorni ko‘rsatadi.
- `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub -E sha256`:
  - `-l`: kalit haqidagi ma’lumotni (barmoq izini) ko‘rsatish.
  - `-f`: o‘qiladigan kalit fayl yo‘li.
  - `-E sha256`: barmoq izini hisoblashda SHA256 xeshidan foydalanish.
  - `tee`: Natijani ham ekranda ko‘rsatadi, ham faylga yozib qo‘yadi. **Ushbu SHA256 barmoq izini eslab qoling!**

---

### 3-BOSQICH: Ubuntu Hostda Ed25519 Kalitini Yaratish va Joylashtirish

#### 3.1. Ubuntu Hostda shaxsiy va ochiq kalit juftligini yaratish
Diqqat! Bu ish sizning asosiy **Ubuntu tizimingiz** terminalida bajariladi:

```bash
mkdir -p ~/A11 ~/.ssh
chmod 700 ~/.ssh

KEY="$HOME/.ssh/a11_ed25519"
KNOWN="$HOME/.ssh/a11_known_hosts"

if [ -f "$KEY" ]; then
    echo "A11 kaliti mavjud, qayta yozilmaydi."
else
    ssh-keygen -t ed25519 -f "$KEY" -C "A11-labssh"
fi

cp "$KEY.pub" ~/A11/a11_ed25519.pub
ssh-keygen -lf "$KEY.pub" -E sha256
```
*(Kalit yaratilayotganda `passphrase` (parolli himoya) so‘raladi, unga maxfiy so‘z bering!)*

🔍 **CLI buyruqlari tahlili:**
- `chmod 700 ~/.ssh`: `.ssh` papkasiga faqat joriy foydalanuvchiga to‘liq ruxsat (`rwx------`) beradi, begona foydalanuvchilar undagi fayllarni o‘qiy olmaydi.
- `ssh-keygen -t ed25519 -f "$KEY" -C "A11-labssh"`:
  - `-t ed25519`: Kalit turini zamonaviy Ed25519 deb belgilash.
  - `-f "$KEY"`: Generatsiya qilingan kalit saqlanadigan fayl yo‘li (`~/.ssh/a11_ed25519` - maxfiy kalit, `~/.ssh/a11_ed25519.pub` - ochiq kalit).
  - `-C "A11-labssh"`: Kalit oxiriga tushunarli izoh qo‘shish.
- `ssh-keygen -lf "$KEY.pub" -E sha256`: Ochiq kalitning SHA256 barmoq izini chiqaradi. Bu barmoq izi serverdagi `authorized_keys` bilan bir xil bo‘lishi kerak.

---

#### 3.2. Ochiq kalitni (`.pub`) SSH-SRV ga xavfsiz o‘tkazish
Ochiq kalit matnini ko‘rib olamiz:
Ubuntu Host terminalida:
```bash
cat ~/A11/a11_ed25519.pub
```
*(Ekranga chiqqan `ssh-ed25519 AAAA... A11-labssh` ko‘rinishidagi bir qatorlik matnni to‘liq nusxalab oling).*

Endi **SSH-SRV konsoliga** o‘tib, ushbu ochiq kalitni `labssh` hisobining `authorized_keys` fayliga qat’iy xavfsizlik ruxsatlari bilan yozamiz:

```bash
sudo -i
install -d -m 700 -o labssh -g labssh /home/labssh/.ssh

# Quyidagi o'zgaruvchiga Ubuntu Hostdan nusxalangan ochiq kalit matnini qo'ying:
PUB_KEY='<UBUNTU_HOSTDAN_NUSXALANGAN_PUB_KALIT>'

echo "$PUB_KEY" > /home/labssh/.ssh/authorized_keys
chown labssh:labssh /home/labssh/.ssh/authorized_keys
chmod 600 /home/labssh/.ssh/authorized_keys

ssh-keygen -lf /home/labssh/.ssh/authorized_keys -E sha256
```

🔍 **CLI buyruqlari tahlili:**
- `install -d -m 700 -o labssh -g labssh /home/labssh/.ssh`:
  - `-d`: Yangi katalog yaratish.
  - `-m 700`: Katalog huquqini `rwx------` (faqat egasiga ruxsat) qilib o‘rnatish.
  - `-o labssh`: Egasi (owner) `labssh`.
  - `-g labssh`: Guruhi (group) `labssh`.
- `chmod 600 /home/labssh/.ssh/authorized_keys`: Fayl huquqini `rw-------` (faqat egasi o‘qishi va yozishi mumkin) qiladi. **Muhim xavfsizlik qoidasi:** Agar `authorized_keys` yoki `.ssh` papkasida boshqalarga yozish ruxsati (`write`) bo‘lsa, OpenSSH server xavfsizlik nuqtai nazaridan kalit bilan kirishni butunlay rad etadi!
- `ssh-keygen -lf ...`: Serverga yozilgan kalit barmoq izini tekshiradi (Hostdagi barmoq izi bilan 100% bir xil bo‘lishi shart).

---

#### 3.3. Cheklov kiritishdan oldin kalit bilan kirishni sinash
Cheklovchi parametrlarni yoqishdan oldin kalit bilan ulanish ishlayotganini sinab ko‘rish shart.

**Ubuntu Host terminalida:**
```bash
ssh -i ~/.ssh/a11_ed25519 \
    -o IdentitiesOnly=yes \
    -o UserKnownHostsFile=~/.ssh/a11_known_hosts \
    -o HostKeyAlgorithms=ssh-ed25519 \
    labssh@10.110.10.10 'whoami; hostname; id'
```

🔍 **CLI bayroqlari tahlili:**
- `ssh`: Secure Shell mijozi.
- `-i ~/.ssh/a11_ed25519`: Autentifikatsiya uchun aynan qaysi shaxsiy kalit identifikatoridan foydalanishni ko‘rsatadi.
- `-o IdentitiesOnly=yes`: SSH agentidagi barcha begona kalitlarni yubormay, faqat `-i` orqali ko‘rsatilgan kalitni yuborishga majburlaydi.
- `-o UserKnownHostsFile=~/.ssh/a11_known_hosts`: Tizimning umumiy `~/.ssh/known_hosts` faylini ifloslantirmaslik uchun laboratoriya uchun alohida hostlar ro‘yxatidan foydalanadi.
- `-o HostKeyAlgorithms=ssh-ed25519`: Serverdan faqat Ed25519 turidagi host kalitini qabul qilishni belgilaydi.
- `labssh@10.110.10.10`: Ulanilayotgan foydalanuvchi va server IP manzili.
- `'whoami; hostname; id'`: Masofaviy serverda interaktiv sessiya ochmasdan bajariladigan test buyruqlari.

⚠️ **Kutiladigan natija:**
1. Birinchi ulanishda server barmoq izini tasdiqlash so‘raladi:
   `The authenticity of host '10.110.10.10' can't be established. ED25519 key fingerprint is SHA256:.... Are you sure you want to continue connecting (yes/no/[fingerprint])?`
   **DIQQAT:** Ushbu barmoq izi 2.1-bosqichda server konsolida ko‘rgan barmoq izimiz bilan bir xil ekanligini tasdiqlang! Mos kelsa, `yes` deb yozing.
2. `Enter passphrase for key`: Shaxsiy kalitingiz parolini kiriting.
3. Ekranda `labssh`, server nomi va uning UID/GID ma’lumotlari chiqadi. Kalit muvaffaqiyatli ishladi!

---

### 4-BOSQICH: SSH Xizmatini Qat’iylashtirish (Hardening) va UFW Sozlash

Kalit ishlagani tasdiqlangach, endi zaif usullarni (parol, root, tunnel) to‘liq bloklaymiz.

#### 4.1. SSH-SRV da cheklovchi konfiguratsiya fayli yaratish
OpenSSH da konfiguratsiya `/etc/ssh/sshd_config.d/` katalogidagi `.conf` fayllar orqali kengaytiriladi.

**SSH-SRV konsolida:**
```bash
sudo -i
cat > /etc/ssh/sshd_config.d/00-a11-lab.conf <<'CONF'
PubkeyAuthentication yes
AuthenticationMethods publickey
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
AllowUsers labssh
MaxAuthTries 3
LoginGraceTime 30
DisableForwarding yes
X11Forwarding no
PermitTunnel no
LogLevel VERBOSE
CONF
```

Konfiguratsiyani tekshiramiz va samaradorligini tahlil qilamiz:
```bash
/usr/sbin/sshd -t
/usr/sbin/sshd -T -C user=labssh,host=admin,addr=10.110.10.1 > /root/A11/sshd-effective.txt
grep -E 'authenticationmethods|passwordauthentication|permitrootlogin|allowusers|disableforwarding' /root/A11/sshd-effective.txt
```

🔍 **Konfiguratsiya parametrlari va buyruqlar tahlili:**
- `PubkeyAuthentication yes`: Kriptografik ochiq kalitlar orqali autentifikatsiyani yoqadi.
- `AuthenticationMethods publickey`: **Eng muhim parametr!** Tizimga kirish uchun faqat va faqat ochiq kalit talab qilinishini qat’iy belgilaydi.
- `PasswordAuthentication no`: Standart parollar orqali kirishni to‘liq taqiqlaydi.
- `KbdInteractiveAuthentication no`: Klaviaturadan interaktiv javob-so‘rov (PAM orqali parol kiritish) usulini o‘chiradi.
- `PermitRootLogin no`: `root` foydalanuvchisiga masofadan SSH orqali kirishni butunlay man etadi.
- `AllowUsers labssh`: Oq ro‘yxat (whitelist) — serverga faqat `labssh` nomli hisob kira oladi. Boshqa foydalanuvchilar hatto to‘g‘ri kalitga ega bo‘lsa ham kiritilmaydi.
- `MaxAuthTries 3`: Bitta ulanish sessiyasida kalitni sinab ko‘rishning maksimal urinishlar soni (bruteforce dan himoya).
- `LoginGraceTime 30`: Agar foydalanuvchi 30 soniya ichida autentifikatsiyadan o‘ta olmasa, ulanish uziladi.
- `DisableForwarding yes`: Port yo‘naltirish (port forwarding) va proksi qilishni bloklaydi.
- `X11Forwarding no`: Grafik interfeyslarni (X11) masofaga uzatishni taqiqlaydi.
- `PermitTunnel no`: SSH orqali virtual tarmoq tunnellari (tun/tap) yaratishni taqiqlaydi.
- `LogLevel VERBOSE`: Xavfsizlik hodisalarini, ayniqsa qaysi kalit bilan kirilgani yoki rad etilganini batafsil loglarga yozadi.
- `/usr/sbin/sshd -t`: Konfiguratsiya sintaksisini tekshirish. Xato bo‘lmasa hech narsa chiqarmaydi (exit code 0).
- `/usr/sbin/sshd -T -C user=labssh,host=admin,addr=10.110.10.1`: Test rejimida amaldagi barcha parametrlarni hisoblab chiqadi. `-C` (connection context) orqali `10.110.10.1` dan kelgan `labssh` ulanishi uchun aynan qaysi qoidalar kuchga kirishini ko‘rsatadi.

---

#### 4.2. SSH-SRV da UFW Firewall bilan manba cheklovini o‘rnatish
Faqat bizning boshqaruv hostimiz `10.110.10.1` ga 22-portni ochamiz, qolgan barcha manbalar yopiladi.

**SSH-SRV konsolida:**
```bash
ufw status numbered
ufw default deny incoming
ufw default allow outgoing
ufw allow in on "$IF" from 10.110.10.1 to 10.110.10.10 port 22 proto tcp comment 'A11 admin SSH'
ufw --force enable
/usr/sbin/sshd -t && systemctl reload ssh.service
ufw status verbose
```

🔍 **CLI buyruqlari tahlili:**
- `ufw status numbered`: UFW ning mavjud qoidalarini raqamlangan ro‘yxat qilib chiqaradi. Agar avvaldan keng `22/tcp ALLOW IN Anywhere` qoidasi bo‘lsa, uni `ufw delete <raqam>` bilan o‘chirib tashlash shart!
- `ufw default deny incoming`: Sukut bo‘yicha barcha kiruvchi paketlarni bloklash (Default Deny siyosati).
- `ufw default allow outgoing`: Serverdan chiquvchi paketlarga ruxsat berish.
- `ufw allow in on "$IF" from 10.110.10.1 to 10.110.10.10 port 22 proto tcp`:
  - `allow in`: Kiruvchi oqimga ruxsat.
  - `on "$IF"`: Aynan laboratoriya interfeysi orqali.
  - `from 10.110.10.1`: Faqat Ubuntu Host IP manzilidan kelganda.
  - `to 10.110.10.10`: Serverning aniq IP manziliga kelganda.
  - `port 22 proto tcp`: Faqat 22-raqamli TCP portiga.
  - `comment '...'`: Qoidaga tavsif qo‘shish.
- `ufw --force enable`: Xavfsizlik devorini tasdiq so‘ramasdan faollashtiradi va tizim yuklanganda avtomatik ishga tushadigan qiladi.
- `systemctl reload ssh.service`: SSH demonini to‘xtatmasdan yangi konfiguratsiyani kuchga kiritadi.

---

#### 4.3. SSH Parol usuli rad etilishini sinash (Salbiy Sinov)
Ubuntu Hostdan maxsus buyruq berib, kalitni o‘chirib, parolni majburlab ko‘ramiz:

**Ubuntu Host terminalida:**
```bash
ssh -o UserKnownHostsFile=~/.ssh/a11_known_hosts \
    -o PubkeyAuthentication=no \
    -o PreferredAuthentications=password,keyboard-interactive \
    labssh@10.110.10.10
```

🔍 **CLI parametrlari tahlili:**
- `-o PubkeyAuthentication=no`: Mijozga kalitlardan foydalanishni qat’iy taqiqlaydi.
- `-o PreferredAuthentications=password,keyboard-interactive`: Serverdan faqat parol va interaktiv usullarni talab qilishni so‘raydi.

⚠️ **Kutiladigan natija:**
```
labssh@10.110.10.10: Permission denied (publickey).
```
Server hatto parolni so‘ramaydi ham! U darhol faqat `publickey` qabul qilinishini aytib ulanishni uzadi. Bu parol bilan bruteforce qilish imkoniyati 100% yo‘q qilinganini isbotlaydi.

---

### 5-BOSQICH: Windows 11 VM da RDP Xizmatini Sozlash, NLA, TLS va Guruh Siyosatlari (RDP-SRV)

Ushbu bosqich **Windows 11 Desktop VM konsolida**, **Administrator PowerShell** oynasida bajariladi.

#### 5.1. Foydalanuvchilarni yaratish va imtiyozlarni ajratish (Least Privilege)
Ikkita hisob yaratamiz:
1. `labrdp` — RDP ga kirish huquqi bor oddiy foydalanuvchi.
2. `labdeny` — To‘g‘ri paroli bor, lekin RDP huquqi berilmagan sinov foydalanuvchisi.

**Windows PowerShell (Admin):**
```powershell
$p1 = Read-Host 'labrdp uchun parol kiriting' -AsSecureString
New-LocalUser -Name labrdp -Password $p1 -Description 'A11 RDP user'

$p2 = Read-Host 'labdeny uchun boshqa parol kiriting' -AsSecureString
New-LocalUser -Name labdeny -Password $p2 -Description 'A11 deny test'

# SID orqali foydalanuvchilar (Users) va RDP foydalanuvchilari (Remote Desktop Users) guruhlarini aniqlash
$usersGroup = (Get-LocalGroup -SID 'S-1-5-32-545').Name
$rduGroup = (Get-LocalGroup -SID 'S-1-5-32-555').Name

Add-LocalGroupMember -Group $usersGroup -Member labrdp,labdeny
Add-LocalGroupMember -Group $rduGroup -Member labrdp

# Tekshirish:
Get-LocalGroupMember -Group $rduGroup
Get-LocalGroupMember -SID 'S-1-5-32-544' # Administrators guruhi
```

🔍 **PowerShell buyruqlari tahlili:**
- `Read-Host -AsSecureString`: Parolni ekranda yashirib, xotirada xavfsiz shifrlangan holatda qabul qiladi.
- `New-LocalUser`: Windows tizimida yangi mahalliy hisob yaratadi.
- `SID 'S-1-5-32-555'`: Bu **Remote Desktop Users** guruhining xalqaro xavfsizlik identifikatori (SID). Windows tili ruscha, inglizcha yoki boshqa tilda bo‘lishidan qat’i nazar, SID o‘zgarmaydi.
- `Add-LocalGroupMember`: `labrdp` foydalanuvchisini masofaviy kirish guruhiga qo‘shadi, `labdeny` esa bu guruhga qo‘shilmaydi.
- `SID 'S-1-5-32-544'`: **Administrators** guruhi. Ikkala hisob ham bu guruhda bo‘lmasligi shart (imtiyozlarni cheklash tamoyili).

---

#### 5.2. Windows Firewall tor ruxsat qoidasini sozlash
Standart barcha 3389 port qoidalarini o‘chirib, faqat Ubuntu Host (`10.110.10.1`) ga ruxsat beramiz.

**Windows PowerShell (Admin):**
```powershell
$labNic = (Get-NetAdapter | Where-Object Status -eq 'Up')[0].Name

# Standart keng RDP qoidalarini o'chirish
Get-NetFirewallRule -Name 'RemoteDesktop*' -ErrorAction SilentlyContinue | Disable-NetFirewallRule

# Standart kiruvchi oqimni barcha profillarda bloklash
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True -DefaultInboundAction Block

# Faqat 10.110.10.1 ga ruxsat beruvchi tor qoida yaratish
New-NetFirewallRule -Name 'A11-RDP-Admin' `
    -DisplayName 'A11 RDP Admin Only' `
    -Direction Inbound `
    -Action Allow `
    -Protocol TCP `
    -LocalPort 3389 `
    -LocalAddress 10.110.10.20 `
    -RemoteAddress 10.110.10.1 `
    -InterfaceAlias $labNic `
    -Profile Any

# Qoidani tekshirish:
Get-NetFirewallRule -Name 'A11-RDP-Admin' | Get-NetFirewallAddressFilter
```

🔍 **PowerShell buyruqlari tahlili:**
- `Disable-NetFirewallRule`: Windows dagi barcha keng `RemoteDesktop*` qoidalarini faolsizlantiradi.
- `Set-NetFirewallProfile -DefaultInboundAction Block`: Xavfsizlik devori qoidalariga mos kelmagan barcha kiruvchi ulanishlarni so‘zsiz bloklash rejimiga o‘tkazadi.
- `New-NetFirewallRule`:
  - `-Direction Inbound`: Kiruvchi paketlar uchun.
  - `-Action Allow`: Ruxsat berish.
  - `-Protocol TCP -LocalPort 3389`: RDP protokoli TCP 3389 porti.
  - `-LocalAddress 10.110.10.20`: Paketlar serverning aynan shu IP siga kelganda.
  - `-RemoteAddress 10.110.10.1`: **Eng muhim cheklov!** Paket faqat va faqat Ubuntu Host (`10.110.10.1`) dan kelgan bo‘lishi shart. Boshqa har qanday IP rad etiladi.

---

#### 5.3. RDP, NLA, TLS va Ma’lumot Chiqib Ketishiga Qarshi Siyosatni (Clipboard/Drive) Sozlash
Bu sozlamalar Windows Registry va Group Policy orqali kiritiladi:

**Windows PowerShell (Admin):**
```powershell
$ts = 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server'
$pol = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services'

# RDP xizmatini yoqish (0 = ulanishlarga ruxsat berilgan)
Set-ItemProperty -Path $ts -Name fDenyTSConnections -Value 0

# Terminal Services siyosat katalogini yaratish
New-Item -Path $pol -Force | Out-Null

# 1. NLA ni majburiy qilish (UserAuthentication = 1)
New-ItemProperty -Path $pol -Name UserAuthentication -PropertyType DWord -Value 1 -Force | Out-Null

# 2. Xavfsizlik qatlamini TLS (SSL) qilish (SecurityLayer = 2)
New-ItemProperty -Path $pol -Name SecurityLayer -PropertyType DWord -Value 2 -Force | Out-Null

# 3. Clipboard (bufer) almashishni taqiqlash (fDisableClip = 1)
New-ItemProperty -Path $pol -Name fDisableClip -PropertyType DWord -Value 1 -Force | Out-Null

# 4. Qattiq disklarni sessiyaga ulashni taqiqlash (fDisableCdm = 1)
New-ItemProperty -Path $pol -Name fDisableCdm -PropertyType DWord -Value 1 -Force | Out-Null

# Muvaffaqiyatli va muvaffaqiyatsiz kirishlarni audit qilishni yoqish
auditpol /set /subcategory:"{0cce9215-69ae-11d9-bed3-505054503030}" /success:enable /failure:enable

# Siyosatlarni zudlik bilan yangilash va tizimni qayta yuklash
gpupdate /target:computer /force
Restart-Computer
```

🔍 **Parametrlar tahlili:**
- `fDenyTSConnections = 0`: Windows tizimida Masofaviy ish stoli (RDP) ulanishlarini qabul qilishga ruxsat beradi.
- `UserAuthentication = 1`: **Network Level Authentication (NLA)** ni majburiy qiladi. NLA qo‘llab-quvvatlamaydigan mijozlar rad etiladi.
- `SecurityLayer = 2`: Ulanishni to‘liq TLS (SSL) shifrlash protokoli orqali amalga oshirishni talab qiladi (Native RDP shifrlash o‘chiriladi).
- `fDisableClip = 1`: Foydalanuvchi host kompyuteri va server o‘rtasida matn yoki fayllarni nusxalab ololmaydi.
- `fDisableCdm = 1`: Mijozning mahalliy disklari serverga virtual disk sifatida ulanmaydi (fayl o‘g‘irlash va ransomware tarqalishining oldi olinadi).
- `auditpol /set /subcategory:"Logon" ...`: Tizimga kirish (Logon) va rad etilgan urinishlarni Windows Security loglarida (Hodisalar 4624 va 4625) qayd etishni boshlaydi.
- `Restart-Computer`: Barcha xizmatlar va registr sozlamalari toza yuklanishi uchun virtual mashinani qayta yuklaydi.

---

#### 5.4. Qayta yuklangach amaldagi holatni tekshirish
Windows qayta yongach, konsolga Administrator bilan kiring va tekshiring:

**Windows PowerShell (Admin):**
```powershell
$pol = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services'
Get-ItemProperty $pol | Select-Object UserAuthentication,SecurityLayer,fDisableClip,fDisableCdm

# RDP tinglovchisi holatini tekshirish
Get-CimInstance -Namespace root/cimv2/terminalservices -ClassName Win32_TSGeneralSetting -Filter "TerminalName='RDP-tcp'" |
    Select-Object TerminalName,UserAuthenticationRequired,SecurityLayer,SSLCertificateSHA1Hash

# 3389 port tinglanayotganini tekshirish
Get-NetTCPConnection -State Listen -LocalPort 3389

# Sertifikat ma'lumotlarini faylga saqlash
Get-ChildItem 'Cert:\LocalMachine\Remote Desktop' |
    Select-Object Subject,Thumbprint,NotBefore,NotAfter |
    Format-List | Out-File C:\A11\rdp-certificate.txt

Get-Content C:\A11\rdp-certificate.txt
```
Kutiladigan natijalar: `UserAuthentication: 1`, `SecurityLayer: 2`, `fDisableClip: 1`, `fDisableCdm: 1`.
Sertifikat barmoq izini (`Thumbprint`) ko‘rib oling.

---

### 6-BOSQICH: Ubuntu Hostdan RDP Ulanish Sinovlari

Endi barcha RDP sinovlarini o‘zimizning **Ubuntu Hostimiz** orqali bajaramiz. Linux muhitida eng qulay va professional RDP mijozi bu `xfreerdp` (FreeRDP) hisoblanadi.

#### 6.1. Ruxsatli foydalanuvchi (`labrdp`) bilan kirish va siyosatlarni sinash
Ubuntu host terminalida:

```bash
xfreerdp /v:10.110.10.20 \
         /u:labrdp \
         /sec:nla \
         +clipboard \
         /drive:testshare,/tmp \
         /cert:ignore
```
*(Parol so‘ralganda `labrdp` uchun qo‘ygan parolingizni kiriting).*

🔍 **CLI bayroqlari tahlili:**
- `xfreerdp`: FreeRDP mijozi buyrug‘i.
- `/v:10.110.10.20`: Server IP manzili.
- `/u:labrdp`: Foydalanuvchi nomi.
- `/sec:nla`: Majburiy NLA protokoli orqali ulanish.
- `+clipboard`: Mijoz bufer almashishni yoqishni so‘raydi.
- `/drive:testshare,/tmp`: Mijoz o‘zining `/tmp` katalogini serverga disk qilib ulashni so‘raydi.
- `/cert:ignore`: O‘z-o‘zini imzolagan laboratoriya sertifikati ogohlantirishini e’tiborsiz qoldirish.

🧪 **Siyosat sinovlari (RDP oynasi ichida):**
1. Masofaviy ish stoli muvaffaqiyatli ochiladi. Terminal (PowerShell yoki cmd) ochib `whoami` deb yozing: natija `kompyuter_nomi\labrdp` bo‘ladi.
2. **Clipboard sinovi:** Ubuntu hostdagi matn muharriridan ixtiyoriy matn nusxalang (masalan: `A11-clipboard-test`). RDP ichidagi Notepad dasturiga uni `Ctrl+V` qilib qo‘yishga urining. **Natija: Matn o‘tmaydi!** Chunki serverdagi `fDisableClip = 1` siyosati bufer kanalini to‘sib qo‘ygan.
3. **Disk yo‘naltirish sinovi:** RDP ichida `This PC` (Mening kompyuterim) ni oching. Ubuntu hostdan yo‘naltirilgan `testshare` nomli tarmoq diski **ko‘rinmaydi!** Chunki `fDisableCdm = 1` siyosati disk drayverini yuklashga ruxsat bermaydi.

Sinov tugagach, RDP oynasidan **Sign out** qiling.

---

#### 6.2. Ruxsatsiz foydalanuvchi (`labdeny`) bilan kirish (Huquq cheklovi sinovi)
Endi `labdeny` foydalanuvchisi bilan ulanib ko‘ramiz. Eslatma: bu foydalanuvchining **paroli to‘g‘ri**, lekin unga RDP ruxsati berilmagan!

**Ubuntu Host terminalida:**
```bash
xfreerdp /v:10.110.10.20 /u:labdeny /sec:nla /cert:ignore
```
*(Parol so‘ralganda `labdeny` uchun berilgan to‘g‘ri parolni kiriting).*

⚠️ **Kutiladigan natija:**
NLA autentifikatsiyasidan so‘ng server ulanishni darhol to‘xtatadi va quyidagi xatolikni qaytaradi:
```
Authentication only, exit status 0 / Connection rejected by server: User account not authorized for remote desktop
```
Bu minimal imtiyoz tamoyili to‘liq ishlayotganini va oddiy foydalanuvchilar o‘z-o‘zidan serverga masofadan kira olmasligini isbotlaydi.

---

### 7-BOSQICH: TEST-VM dan Ruxsatsiz Manba Sinovlari (Firewall Tekshiruvi)

Ushbu bosqichda begona **TEST-VM (`10.110.10.30`)** virtual mashinasidan turib, ikkala serverning portlariga ulanishga harakat qilamiz.

**TEST-VM konsolida:**
```bash
ip -br addr
nc -vz -w 3 10.110.10.10 22
nc -vz -w 3 10.110.10.20 3389
```

🔍 **CLI buyruqlari tahlili:**
- `ip -br addr`: TEST-VM ning IP manzili haqiqatan ham `10.110.10.30` ekanligini tasdiqlaydi.
- `nc`: Netcat dasturi (tarmoq ulanishlarini tekshirish vositasi).
- `-v`: Verbose (batafsil ma’lumot chiqarish).
- `-z`: Zero-I/O rejimi (ma’lumot uzatmaydi, faqat TCP port ochilishini (3-way handshake) tekshirib darhol yopadi).
- `-w 3`: Timeout vaqti 3 soniya (agar 3 soniyada javob kelmasa to‘xtaydi).
- `10.110.10.10 22`: SSH serverning 22-porti.
- `10.110.10.20 3389`: RDP serverning 3389-porti.

⚠️ **Kutiladigan natija:**
```
nc: connect to 10.110.10.10 port 22 (tcp) timed out: Operation now in progress
nc: connect to 10.110.10.20 port 3389 (tcp) timed out: Operation now in progress
```
Ikkala ulanish ham `timed out` bilan tugaydi! Paketlar serverlarga yetib bormasdan xavfsizlik devorlari (UFW va Windows Firewall) tomonidan jimgina tashlab yuboriladi (`DROP`).

Darhol Ubuntu Hostdan tekshiramiz:
```bash
nc -vz 10.110.10.10 22
nc -vz 10.110.10.20 3389
```
Ubuntu hostda ikkala port ham `open` (muvaffaqiyatli) bo‘ladi. Bu xizmatlar o‘chib qolmaganini, balki TEST-VM aynan firewall manba qoidasi bilan bloklanganini isbotlaydi.

---

### 8-BOSQICH: Tizim Loglarini Yig‘ish va Xavfsizlik Auditi

#### 8.1. SSH-SRV loglarini tekshirish
SSH-SRV konsolida:
```bash
sudo journalctl -u ssh.service --since '90 minutes ago' --no-pager > /root/A11/ssh-journal.txt
grep -E 'Accepted publickey|Failed publickey|Connection closed' /root/A11/ssh-journal.txt
ufw status numbered > /root/A11/ufw-status.txt
```

🔍 **Log tahlili:**
- `Accepted publickey for labssh from 10.110.10.1 port ... ssh2: ED25519 SHA256:...`: Bu yozuv `labssh` hisobiga aynan `10.110.10.1` dan va to‘g‘ri kalit barmoq izi orqali kirilganini isbotlaydi.
- Parol bilan ulanishga urinilganda esa hech qanday "Accepted password" yo‘qligi va ulanish darhol uzilgani ko‘rinadi.
- TEST-VM ning ulanishlari esa UFW tomonidan to‘xtatilgani sababli `sshd` loglariga yetib ham kelmaydi.

---

#### 8.2. Windows RDP-SRV hodisalarini tekshirish (Event Viewer)
Windows RDP-SRV konsolida:
```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4624, 4625
    StartTime = (Get-Date).AddMinutes(-90)
} -ErrorAction SilentlyContinue |
Select-Object TimeCreated, Id, Message |
Format-List | Out-File C:\A11\logon-events.txt

Get-Content C:\A11\logon-events.txt | Select-String -Pattern 'labrdp|labdeny|Logon Type' -Context 0,2
```

🔍 **Hodisalar tahlili:**
- **Hodisa 4624 (An account was successfully logged on):**
  - Foydalanuvchi: `labrdp`
  - **Logon Type:** `10` (RemoteInteractive — RDP orqali kirishni anglatadi).
  - Source Network Address: `10.110.10.1`.
- **Hodisa 4625 (An account failed to log on) yoki Kirish rad etilishi:**
  - Foydalanuvchi: `labdeny`. Tizim uning paroli to‘g‘riligini tasdiqlagan, lekin ruxsat huquqi yo‘qligi sababli sessiya berilmagan.

---

## 4. 8 TA ASOSIY SINOVNING NATIJALAR JADVALI (TEST MATRIX)

Hisobot va o‘qituvchiga amaliy himoya qilish uchun ushbu natijalar jadvali asosiy dalil bo‘lib xizmat qiladi:

| № | Sinov Nomi va Yo‘nalishi | Kutilgan Natija | Haqiqiy Natija va Dalil | Xavfsizlik Isboti |
| :---: | :--- | :--- | :--- | :--- |
| **1** | **Host → SSH**, to‘g‘ri Ed25519 kaliti bilan | `labssh` muvaffaqiyatli kiradi | Muvaffaqiyatli ulandi; `whoami` -> `labssh`, `id` -> `uid=1001(labssh)` | Kalitli autentifikatsiya to‘g‘ri sozlangan, ruxsatlar xatosiz. |
| **2** | **Host → SSH**, faqat parol usuli majburlanganda | Ulanish rad etiladi, kalit talab qilinadi | `Permission denied (publickey)` | Parol bilan buzib kirish (bruteforce) imkoniyati 100% yo‘q qilingan. |
| **3** | **TEST-VM → SSH:22**, tarmoq port sinovi | TCP ulanishi o‘tmaydi (Timeout) | `nc: connect to 10.110.10.10 port 22 timed out` | UFW faqat `10.110.10.1` ga ruxsat bergan, begona manbalar bloklangan. |
| **4** | **Host → RDP**, `labrdp` hisobi bilan | NLA dan o‘tib, oddiy foydalanuvchi sessiyasi ochiladi | RDP ish stoli ochildi; `whoami` -> `labrdp` | NLA to‘g‘ri ishlamoqda, `Remote Desktop Users` huquqi mavjud. |
| **5** | **Host → RDP**, `labdeny` + to‘g‘ri parol bilan | Autentifikatsiyadan so‘ng kirish huquqi yo‘qligi sabab rad etiladi | `User account not authorized for remote desktop` | Minimal imtiyoz tamoyili ishlayapti, ortiqcha hisoblar bloklangan. |
| **6** | **TEST-VM → RDP:3389**, tarmoq port sinovi | TCP ulanishi o‘tmaydi (Timeout) | `nc: connect to 10.110.10.20 port 3389 timed out` | Windows Firewall begona manbalardan 3389 portni to‘liq himoyalagan. |
| **7** | **RDP Clipboard sinovi**, Host matnini RDP ga qo‘yish | Matn bufer orqali o‘tmaydi | Hostdan nusxalangan matn RDP Notepadga qo‘yilmadi (`Paste` faol emas) | `fDisableClip = 1` siyosati orqali ma’lumotlar oqishi bartaraf etilgan. |
| **8** | **RDP Disk sinovi**, Host papkasini disk qilib ulash | Host diski RDP ichida ko‘rinmaydi | `This PC` ichida yo‘naltirilgan disklar umuman yo‘q | `fDisableCdm = 1` siyosati orqali fayl o‘g‘irlash va virus tarqalishi to‘silgan. |

---

## 5. NOSOZLIKLARNI ANIQLASH VA BARTARAF ETISH (TROUBLESHOOTING)

Laboratoriya davomida yuzaga kelishi mumkin bo‘lgan tipik xatolar va ularning yechimlari:

### 1. SSH: "Permission denied (publickey)"
- **Sababi:** Serverdagi `.ssh` papkasi yoki `authorized_keys` faylining huquqlari noto‘g‘ri. Agar ruxsat `777` yoki boshqa foydalanuvchilar yoza oladigan bo‘lsa, OpenSSH ularni o‘qishdan bosh tortadi.
- **Yechim:** Server konsolida huquqlarni to‘g‘rilang:
  ```bash
  chmod 700 /home/labssh/.ssh
  chmod 600 /home/labssh/.ssh/authorized_keys
  chown -R labssh:labssh /home/labssh/.ssh
  ```

### 2. "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!"
- **Sababi:** Virtual mashina qayta o‘rnatilgan yoki yangi snapshot yuklangan, natijada serverning Host kaliti o‘zgargan, lekin Ubuntu hostingizdagi `a11_known_hosts` da eski kalit saqlanib qolgan.
- **Yechim:** Host konsolidan server barmoq izini tasdiqlang va eski yozuvni o‘chiring:
  ```bash
  ssh-keygen -f ~/.ssh/a11_known_hosts -R 10.110.10.10
  ```

### 3. SSH konfiguratsiyasi noto‘g‘ri bo‘lib, serverga ulanib bo‘lmay qolsa (Konsoldan tiklash)
- **Sababi:** `00-a11-lab.conf` faylida sintaktik xato ketgan yoki port/foydalanuvchi noto‘g‘ri yozilgan.
- **Yechim:** VMware oynasidan SSH-SRV konsoliga kiring:
  ```bash
  sudo rm /etc/ssh/sshd_config.d/00-a11-lab.conf
  sudo systemctl restart ssh.service
  ```
  Zaxiradagi `/root/A11/ssh-before` nusxasidan qayta tiklashingiz ham mumkin.

### 4. Host ham, TEST-VM ham portga ulana olmayapti (Timeout)
- **Sababi:** Ubuntu hostdagi `vmnet1` interfeysi IP manzili o‘chib ketgan yoki adapter noto‘g‘ri tanlangan.
- **Yechim:** Host terminalida tekshiring:
  ```bash
  ip addr show vmnet1
  sudo ip addr replace 10.110.10.1/24 dev vmnet1
  ```
  Virtual mashinalar sozlamasida adapter **VMnet1 (Host-only)** ga ulanganini tekshiring.

---

## 6. NAZORAT SAVOLLARI VA ULARNING CHUQUR ILMIY JAVOBLARI

### 1. SSH host key bilan foydalanuvchi public key o‘rtasidagi farq nima?
**Javob:**
- **SSH host key** — bu serverning o‘ziga tegishli bo‘lgan assimetrik kalit bo‘lib, server shaxsini mijozga isbotlash uchun xizmat qiladi. U mijozni MITM (soxta server o‘rnatib trafikni ushlash) hujumlaridan himoya qiladi va mijozning `known_hosts` faylida saqlanadi.
- **Foydalanuvchi public key** — bu ulanayotgan shaxsning (mijozning) kaliti bo‘lib, uning serverdagi hisobga kirish huquqini tasdiqlash uchun xizmat qiladi va serverdagi `~/.ssh/authorized_keys` faylida saqlanadi.

### 2. Passphrase va server foydalanuvchisi paroli qayerda tekshiriladi?
**Javob:**
- **Passphrase** — bu mijozning kompyuterida (Ubuntu hostda) saqlanayotgan shaxsiy kalitni (`a11_ed25519`) shifrlab turuvchi parol. U faqat **mijoz kompyuterida** lokal ravishda tekshiriladi va hech qachon tarmoqqa uzatilmaydi.
- **Server foydalanuvchisi paroli** — serverning `/etc/shadow` faylida xeshlangan holda saqlanadi va server tomonidagi PAM (Pluggable Authentication Modules) orqali tekshiriladi. Kalitli autentifikatsiya yoqilganda bu parol umuman ishlatilmaydi.

### 3. Nega SSH kaliti bilan kirishni tekshirmasdan parol usuli o‘chirilmaydi?
**Javob:**
Chunki agar ochiq kalit serverga xato joylashtirilgan bo‘lsa, fayl ruxsatlari noto‘g‘ri o‘rnatilgan bo‘lsa yoki kalit juftligi mos kelmasa, parol usulini o‘chirib yuborish ma’murning masofadan kirish huquqini butunlay yo‘qotishiga (lockout) olib keladi. Serverga kirish faqat jismoniy konsol orqali tiklanishi mumkin bo‘lib qoladi.

### 4. `sshd -t` va `sshd -T` nimani tekshiradi?
**Javob:**
- `sshd -t` (test mode) — barcha konfiguratsiya fayllarining (`/etc/ssh/sshd_config` va `sshd_config.d/*.conf`) sintaktik to‘g‘riligini va xatolar yo‘qligini tekshiradi.
- `sshd -T` (extended test) — barcha kiritilgan, meros qilib olingan va sukut bo‘yicha belgilangan parametrlarni birlashtirib, joriy ulanish konteksti (`-C user=...,addr=...`) bo‘yicha **amalda qaysi parametrlar kuchga kirishini** (effective configuration) ekranga chiqaradi.

### 5. Nima uchun tor firewall allow qoidasi keng allow qoidasini bekor qilmaydi?
**Javob:**
Chunki xavfsizlik devorlari (iptables/nftables, UFW, Windows Firewall) qoidalarni birinchi mos kelish tamoyili (first match wins) yoki mantiqiy "YOKI" (OR) tamoyili bo‘yicha baholaydi. Agar tarmoqda `Allow ANY` (hamma uchun ruxsat) qoidasi mavjud bo‘lsa, paket tor qoidaga tushmasa ham, keng qoidaga tushib ruxsat oladi. Tor qoida ishlashi uchun barcha keng qoidalar o‘chirilishi shart.

### 6. RDPda NLA, TLS va MFA vazifalari qanday farqlanadi?
**Javob:**
- **TLS:** Mijoz va server o‘rtasidagi ma’lumotlar oqimini shifrlaydi va server sertifikatining haqiqiyligini tekshiradi (tarmoqda tinglash va MITM dan himoya).
- **NLA:** Foydalanuvchini to‘liq Windows grafik sessiyasi ochilishidan oldin tarmoq darajasida (CredSSP orqali) autentifikatsiya qiladi (DoS va brute-force dan himoya).
- **MFA (Multi-Factor Authentication):** Foydalanuvchidan biladigan narsasidan (parol) tashqari ega bo‘lgan narsasini (OTP kod, token, biometrika) talab qiladi. NLA ning o‘zi MFA emas, u faqat protokol xavfsizligini oshiradi.

### 7. `labdeny` sinovida nima uchun to‘g‘ri parol ishlatiladi?
**Javob:**
Chunki agar noto‘g‘ri parol ishlatilsa, tizim shunchaki "parol xato" deb rad etadi va bu bilan foydalanuvchining RDP huquqi yo‘qligi isbotlanmay qoladi. To‘g‘ri parol kiritilganda esa autentifikatsiya muvaffaqiyatli o‘tadi, lekin tizim: "Siz haqingizda ma’lumot to‘g‘ri, biroq sizda masofadan kirish huquqi (Remote Desktop Users a’zoligi) yo‘q" deb rad etadi. Bu avtorizatsiya chegarasini isbotlaydi.

### 8. O‘z-o‘zini imzolagan sertifikat bilan ishonchli CA sertifikati qanday farqlanadi?
**Javob:**
- **O‘z-o‘zini imzolagan sertifikat (Self-signed):** Serverning o‘zi tomonidan yaratiladi. Trafikni shifrlaydi, lekin uning haqiqiyligini tasdiqlovchi uchinchi tomon yo‘q. Mijozda har doim xavfsizlik ogohlantirishi chiqadi va MITM hujumlariga zaif bo‘ladi (agar thumbprint qo‘lda tekshirilmasa).
- **CA sertifikati (Certificate Authority):** Dunyo bo‘ylab yoki tashkilot ichida tan olingan markaziy sertifikatlash markazi tomonidan imzolanadi. Mijozlar oldindan ushbu CA ga ishonadi, ogohlantirishlar chiqmaydi va server shaxsi avtomatik kafolatlanadi.

### 9. 4624 hodisasining o‘zi RDP sessiyasi ochilganini qachon yetarli ko‘rsatmaydi?
**Javob:**
Windows da har qanday tarmoq ulanishi, fondagi xizmatlar, NLA tekshiruvi yoki SMB so‘rovlari ham 4624 (muvaffaqiyatli kirish) hodisasini yaratadi. RDP sessiyasi aynan ochilganini isbotlash uchun 4624 hodisasi ichidagi **Logon Type** parametri aynan **10 (RemoteInteractive)** ekanligini ko‘rish shart.

### 10. RDP clipboard cheklovi barcha fayl chiqarish kanallarini to‘xtatadimi?
**Javob:**
Yo‘q, clipboard faqat bufer orqali ma’lumot ko‘chirishni to‘xtatadi. Foydalanuvchi ma’lumotlarni chiqarib ketishi mumkin bo‘lgan boshqa kanallar ham bor: Drive Redirection (lokal disklarni ulash), COM/LPT portlar, printer orqali faylga chop etish, tarmoq orqali yuklash (browser/web) va h.k. Shuning uchun `fDisableCdm` va boshqa siyosatlar birgalikda kompleks qo‘llanilishi lozim.

---

## 7. FOYDALANILGAN RASMIY VA ILMIY MANBALAR

1. **Microsoft Learn — Remote Desktop Access and NLA Architecture:**  
   [https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/remote-desktop-allow-access](https://learn.microsoft.com/en-us/windows-server/remote/remote-desktop-services/remotepc/remote-desktop-allow-access)
2. **Ubuntu Server Documentation — OpenSSH Server Hardening & Key Management:**  
   [https://ubuntu.com/server/docs/how-to/security/openssh-server/](https://ubuntu.com/server/docs/how-to/security/openssh-server/)
3. **OpenBSD / OpenSSH Manual Pages — sshd_config(5) Runtime Specifications:**  
   [https://man.openbsd.org/sshd_config](https://man.openbsd.org/sshd_config)
4. **Microsoft Learn — Windows Defender Firewall with Advanced Security & Inbound Rules:**  
   [https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/rules)
5. **Microsoft Learn — ADMX TerminalServer Policy CSP & Session Redirection Controls:**  
   [https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-admx-terminalserver](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-admx-terminalserver)
6. **Microsoft Learn — Remote Desktop Listener Certificate Configurations and SSL Security:**  
   [https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-listener-certificate-configurations](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/remote-desktop-listener-certificate-configurations)
7. **Microsoft Learn — Security Auditing Event 4624 (Logon Types and Remote Interactive Auditing):**  
   [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624)
8. **NIST SP 800-123 — Guide to General Server Security & Remote Administration Guidelines:**  
   [https://csrc.nist.gov/publications/detail/sp/800-123/final](https://csrc.nist.gov/publications/detail/sp/800-123/final)
9. **RFC 8709 — Ed25519 and Modern Cryptographic Implementations for SSH:**  
   [https://datatracker.ietf.org/doc/html/rfc8709](https://datatracker.ietf.org/doc/html/rfc8709)
