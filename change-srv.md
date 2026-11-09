# VMware Klonlangan Serverlarni Sozlash, Ziddiyatlarni Bartaraf Etish va Optimallashtirish Qo'llanmasi

Ushbu texnik qo'llanma VMware Workstation / Player muhitida klonlangan Linux (Ubuntu Server) virtual mashinalarida yuzaga keladigan apparat va dasturiy ziddiyatlarni (MAC/IP to'qnashuvlari, `machine-id`, netplan, foydalanuvchi hisoblari) to'liq bartaraf etish hamda resurslari cheklangan (byudjetli) noutbuklarda laboratoriya stendini maksimal unumdorlik bilan ishlatish uchun mo'ljallangan.

---

## Mundarija
1. [VMware Apparat Sathidagi Ziddiyatlarni Bartaraf Etish](#1-vmware-apparat-sathidagi-ziddiyatlarni-bartaraf-etish)
2. [Tizim Identifikatorlarini Tozalash va Ajratish](#2-tizim-identifikatorlarini-tozalash-va-ajratish)
3. [Netplan Konfiguratsiyasi (`/etc/netplan/00-installer-config.yaml`)](#3-netplan-konfiguratsiyasi-etcnetplan00-installer-configyaml)
4. [Sudo Foydalanuvchi Hisobi va Parolini Xavfsiz Yangilash](#4-sudo-foydalanuvchi-hisobi-va-parolini-xavfsiz-yangilash)
5. [Byudjetli Noutbuklar Uchun Resurslarni Optimallashtirish (Disk, RAM, CPU)](#5-byudjetli-noutbuklar-uchun-resurslarni-optimallashtirish-disk-ram-cpu)
6. [Laboratoriya Serverlari Uchun Tezkor Tekshiruv Cheklisti](#6-laboratoriya-serverlari-uchun-tezkor-tekshiruv-cheklisti)

---

## 1. VMware Apparat Sathidagi Ziddiyatlarni Bartaraf Etish

Virtual mashina (VM) klonlanganda, VMware ko'pincha virtual apparat sozlamalarini (jumladan, MAC manzillarni) avvalgi holaticha nusxalaydi. Natijada bir nechta VM bitta tarmoqda bir xil MAC manzilga ega bo'lib, **L2 (ARP) va L3 (DHCP/IP) to'qnashuvlari** yuzaga keladi.

```
       +-------------------------------------------------------------+
       |               Xost Kompyuter (Host OS)                      |
       +-------------------------------------------------------------+
             |                                             |
   [VMnet1: Host-Only]                             [VMnet8: NAT]
   Subnet: 192.168.57.0/24                         Tashqi Internet & Paketlar
             |                                             |
       +-----+---------------------------------------------+-----+
       |                                                         |
       |  +---------------------+        +---------------------+ |
       |  |     VM-1 (srv1)     |        |     VM-2 (srv2)     | |
       |  | ens33: 192.168.57.10|        | ens33: 192.168.57.20| |
       |  | ens37: DHCP (NAT)   |        | ens37: DHCP (NAT)   | |
       |  +---------------------+        +---------------------+ |
       +---------------------------------------------------------+
```

### 1.1. Yangi Mustaqil MAC Manzillarni Generatsiya Qilish
Har bir klonlangan yangi VM uchun tarmoq kartalarining MAC manzillarini yangilash shart:

1. Virtual mashinani to'liq o'chiring (**Power Off**).
2. VMware interfeysida VM nomini o'ng tugma bilan bosing -> **Settings** (yoki `Ctrl + D`).
3. Chap paneldan **Network Adapter** (birinchi adapter) ni tanlang.
4. O'ng tomondagi **Advanced...** tugmasini bosing.
5. **MAC Address** bo'limida **Generate** tugmasini bosing. Yangi unikal MAC hosil bo'ladi.
6. Agar ikkinchi adapter mavjud bo'lsa (**Network Adapter 2**), uni ham tanlab, **Advanced... -> Generate** amalini bajaring.
7. **OK** tugmasi orqali o'zgarishlarni saqlang.

> [!IMPORTANT]
> Ushbu amal bajarilmasa, tarmoqdagi switch/bridge jadvallarida ARP poisoning/flapping holati yuz beradi va paketlar tasodifiy ravishda u yoki bu serverga adashib keta boshlaydi.

### 1.2. Ikkita Alohida Tarmoq Adapterini (Dual-Homed) To'g'ri Ulash
Xavfsizlik va tarmoq laboratoriyalari uchun har bir serverda 2 ta tarmoq kartasi bo'lishi tavsiya etiladi:

* **1-adapter (`ens33`): Host-Only (Masalan, VMnet1)**
  - Vazifasi: Faqat xost va laboratoriya serverlari o'rtasidagi izolyatsiyalangan ichki tarmoq (masalan, `192.168.57.0/24`).
  - Xususiyati: Tashqi internetga chiqmaydi, xavfsiz audit va penetratsion testlar uchun xizmat qiladi.
* **2-adapter (`ens37`): NAT (Masalan, VMnet8)**
  - Vazifasi: Tashqi internetga chiqish, `apt update`, `apt install` orqali yangi dasturiy ta'minotlarni o'rnatish.
  - Xususiyati: DHCP orqali avtomatik IP oladi va xost orqali internetga chiqadi.

**VMware ichida ulash tartibi:**
1. **VM Settings -> Add... -> Network Adapter -> Finish**.
2. Birinchi adapterni tanlang: **Host-only: A private network shared with the host** (yoki **Custom: VMnet1**).
3. Ikkinchi adapterni tanlang: **NAT: Used to share the host's IP address** (yoki **Custom: VMnet8**).

### 1.3. VMware Advanced Oynasidagi Kechikish Parametri (`Latency ms`)
VMware tarmog'ining ilg'or sozlamalarida tarmoq kechikishini sun'iy simulyatsiya qilish imkoniyati mavjud.
* **VM Settings -> Network Adapter -> Advanced...** bo'limida:
  - **Incoming transfer / Outgoing transfer** cheklovlari bo'sh yoki cheklanmagan bo'lishi kerak.
  - **Latency (ms)** parametri **qat'iy `0`** qilib belgilanishi shart.
* Agar u yerda tasodifan 10-50 ms kiritilgan bo'lsa, serverlar orasidagi SSH ulanish, ping va lokal testlarda tushunarsiz sekinlashuvlar yuzaga keladi.

---

## 2. Tizim Identifikatorlarini Tozalash va Ajratish

Linux tizimlari (ayniqsa Ubuntu/Debian) tarmoq identifikatsiyasi uchun faqat MAC manzilga tayanmaydi. `systemd` arxitekturasida har bir o'rnatish o'zining unikal `machine-id` kodiga ega bo'ladi.

### 2.1. `machine-id` Muammosi va Uni Qayta Generatsiya Qilish
Klonlash natijasida barcha VM larda `/etc/machine-id` bir xil bo'lib qoladi. Zamonaviy DHCP mijozlari (`systemd-networkd`, `dhclient`) DHCP so'rovida `DUID` (DHCP Unique Identifier) sifatida aynan `machine-id` dan foydalanadi. Natijada **DHCP server barcha klonlarga bitta va bir xil IP manzil beradi**.

**Yechim:**
Eski identifikatorni o'chirib, tizimga yangi 32 belgili unikal identifikator yaratish buyruqlarini bajaring:

```bash
# 1. Eski identifikator fayllarini tozalash
sudo rm -f /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id

# 2. Yangi unikal machine-id yaratish
sudo systemd-machine-id-setup

# 3. dbus uchun ramziy bog'lanish (symlink) yaratish
sudo ln -sf /etc/machine-id /var/lib/dbus/machine-id

# 4. Yangi ID yaratilganini tekshirish
cat /etc/machine-id
```

> [!NOTE]
> `cat /etc/machine-id` buyrug'i 32 belgili o'n oltilik (hexadecimal) satr chiqarishi kerak (masalan: `e8b15d03a1fc4084931a7848f10842e3`). Klonlangan har bir serverda bu qiymat har xil bo'lishi shart!

---

### 2.2. Server Nomi (Hostname) va `/etc/hosts` Faylini Yangilash
Har bir server laboratoriyada o'z vazifasiga mos nomga ega bo'lishi lozim (masalan: `srv1`, `srv2`, `srv-attacker`, `srv-db`):

1. **Yangi hostname belgilash:**
   ```bash
   sudo hostnamectl set-hostname srv1
   ```

2. **`/etc/hosts` faylini moslash:**
   Tizimdagi lokal xizmatlar yangi nomni to'g'ri hal qilishi (resolve) uchun `/etc/hosts` ichidagi `127.0.1.1` qatorini yangilang:
   ```bash
   sudo nano /etc/hosts
   ```
   Fayl mazmuni quyidagicha ko'rinishda bo'lishi kerak:
   ```text
   127.0.0.1 localhost
   127.0.1.1 srv1

   # Quyidagi qatorlar IPv6 uchun
   ::1 ip6-localhost ip6-loopback
   fe00::0 ip6-localnet
   ff00::0 ip6-mcastprefix
   ff02::1 ip6-allnodes
   ff02::2 ip6-allrouters
   ```

3. **Terminal seansini yangilash:**
   Serverni qayta yuklamasdan (reboot qilmasdan) yangi hostname buyruq satrida (bash prompt) ko'rinishi uchun quyidagi buyruqni bering:
   ```bash
   exec bash
   ```
   Natijada terminal taklifi `user@eski-nom:~$` dan `user@srv1:~$` ga o'zgaradi.

---

## 3. Netplan Konfiguratsiyasi (`/etc/netplan/00-installer-config.yaml`)

Ubuntu 18.04/20.04/22.04/24.04 tizimlarida tarmoq konfiguratsiyasi Netplan orqali boshqariladi.

### 3.1. Eski Klon Qoldiqlarini Tozalash
O'rnatish vaqtida Netplan konfiguratsiya fayliga o'sha dastlabki mashinaning apparat ma'lumotlari yozib qo'yilgan bo'ladi:
```yaml
# XATOLIKKA SABAB BO'LADIGAN ESKI QATORLAR:
match:
  macaddress: 00:0c:29:ab:cd:ef
set-name: ens33
```
Yangi klon yaratilib, MAC manzil o'zgartirilgach, Netplan ushbu eski MAC manzilni qidiradi va topa olmagach, tarmoq interfeysini ko'tarmaydi (`device not found` yoki interfeys o'chiq qoladi).

> [!WARNING]
> Netplan konfiguratsiya faylidan barcha `match:` va `macaddress:` qatorlarini butunlay olib tashlash shart! Interfeyslarni bevosita ularning tizimdagi nomlari (`ens33`, `ens37`) orqali e'lon qilish lozim.

---

### 3.2. YAML Konfiguratsiya Faylini Yozish
Interfeyslar nomini aniqlash uchun avval buyruq bering:
```bash
ip link show
```
Odatda birinchi adapter `ens33`, ikkinchisi esa `ens37` (yoki `enp0s3`, `enp0s8`) bo'ladi.

Netplan faylini tahrirlash uchun oching:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
*(Agar fayl nomi boshqacha bo'lsa, `/etc/netplan/` ichidagi `.yaml` faylni tahrirlang).*

Quyidagi konfiguratsiyani kiriting:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    # 1-adapter: Host-Only (Laboratoriya ichki tarmog'i - Statik IP)
    ens33:
      dhcp4: no
      addresses:
        - 192.168.57.10/24

    # 2-adapter: NAT (Internetga chiqish va paketlar o'rnatish - Dinamik IP)
    ens37:
      dhcp4: true
      dhcp4-overrides:
        use-dns: true
        use-routes: true
```

#### Laboratoriya Serverlari Uchun Statik IP Taqsimoti namunasi:
* **Server 1 (`srv1`):** `192.168.57.10/24`
* **Server 2 (`srv2`):** `192.168.57.20/24`
* **Server 3 (`srv3`):** `192.168.57.30/24`
* **Server 4 (`srv4`):** `192.168.57.40/24`

> [!CAUTION]
> **YAML Sintaksis Qoidasi:** YAML formatida **TAB tugmasidan foydalanish qat'iyan taqiqlanadi**! Barcha bo'shliqlar faqat **PROBEL (Space)** orqali (har bir pog'ona uchun 2 ta probel) kiritilishi shart.

> [!NOTE]
> **Marshrutlash (Routing) mantig'i:** Nima uchun `ens33` da `gateway4` yoki `routes` ko'rsatilmadi?
> Chunki serverda faqat bitta birlamchi shlyuz (Default Gateway) bo'lishi kerak. Tashqi internetga chiquvchi Default Gateway avtomatik ravishda NAT interfeysidan (`ens37`) DHCP orqali olinadi. Agar `ens33` ga ham shlyuz berilsa, "Routing Conflict" yuzaga keladi va internet yo'qoladi.

---

### 3.3. Sozlamalarni Kuchga Kiritish va Diagnostika

1. **Konfiguratsiyani sinab ko'rish va qo'llash:**
   ```bash
   sudo netplan apply
   ```
   *(Agar xatolik chiqsa: `sudo netplan --debug apply` buyrug'i orqali YAML qatorlaridagi xato probellarni aniqlang).*

2. **IP manzillarni tekshirish:**
   ```bash
   ip -br addr
   ```
   **Kutilayotgan natija:**
   ```text
   lo               UNKNOWN        127.0.0.1/8 ::1/128 
   ens33            UP             192.168.57.10/24 ...
   ens37            UP             192.168.220.130/24 ...
   ```

3. **Marshrutlar jadvalini tekshirish:**
   ```bash
   ip route
   ```
   `default via 192.168.x.2 dev ens37` (NAT interfeys shlyuzi) mavjudligiga ishonch hosil qiling.

4. **Aloqani sinash:**
   ```bash
   # Ichki tarmoqni tekshirish
   ping -c 3 192.168.57.1

   # Tashqi internetni va DNS ni tekshirish
   ping -c 3 google.com
   ```

---

## 4. Sudo Foydalanuvchi Hisobi va Parolini Xavfsiz Yangilash

Klonlangan serverda ko'pincha eski foydalanuvchi nomi qoladi (masalan, `student`, `ubuntu`, `user1`). Uni yangilashda keng tarqalgan muammo: **foydalanuvchi tizimda ochiq seansda turganida o'z hisob nomini o'zgartira olmaydi** va quyidagi xatolik yuzaga keladi:
```text
usermod: user <eski_nom> is currently used by process <PID>
```

Ushbu muammoni to'g'ri va xavfsiz chetlab o'tish tartibi quyidagicha:

### 4.1. Bosqichma-bosqich Yangilash Jarayoni

#### 1-qadam: Vaqtinchalik root parolini yoqish
Hozirgi foydalanuvchi seansida turib, `root` akkauntiga vaqtinchalik parol bering:
```bash
sudo passwd root
```
*(Yangi root parolini 2 marta kiriting).*

#### 2-qadam: Foydalanuvchi seansidan chiqish
Barcha ochiq dasturlarni yopib, joriy foydalanuvchi hisobidan to'liq chiqing:
```bash
exit
```
*(Agar SSH orqali ulangan bo'lsangiz seans uziladi. VMware konsol oynasiga (`tty1` yoki GUI login ekraniga) o'ting).*

#### 3-qadam: Bevosita `root` sifatida tizimga kirish
VMware oynasida login so'ralganda:
* **Login:** `root`
* **Parol:** 1-qadamda kiritilgan root paroli.

#### 4-qadam: Foydalanuvchi nomi va uy katalogini o'zgartirish
Endi eski foydalanuvchining hech qanday jarayoni (process) xotirada ishlamayotganligi sababli, quyidagi buyruqni xatosiz bajaramiz:
```bash
# Sintaksis: usermod -l <yangi_nom> -d /home/<yangi_nom> -m <eski_nom>
usermod -l adminlab -d /home/adminlab -m student
```
* `-l adminlab`: Yangi login nomini belgilaydi.
* `-d /home/adminlab`: Uy katalogi (home directory) yangi yo'lini belgilaydi.
* `-m`: Eski katalogdagi barcha fayllarni yangi katalogga avtomatik ko'chiradi.

#### 5-qadam: Birlamchi guruh nomini ham yangilash
Foydalanuvchi nomiga mos guruh nomini ham yangilab qo'yish maqsadga muvofiq:
```bash
groupmod -n adminlab student
```

#### 6-qadam: Yangi foydalanuvchiga yangi parol belgilash
```bash
passwd adminlab
```

#### 7-qadam: Huquqlar va sudo guruhini tasdiqlash
Yangi foydalanuvchi `sudo` ma'muriy guruhida saqlanib qolganini tekshiring:
```bash
id adminlab
```
**Kutilayotgan natija:**
```text
uid=1000(adminlab) gid=1000(adminlab) groups=1000(adminlab),4(adm),24(cdrom),27(sudo),...
```

#### 8-qadam: (Xavfsizlik) Root parolini yana qulflash
Ish tugagach, `root` hisobini to'g'ridan-to'g'ri kirishdan yana himoyalab qo'yish xavfsizlik standarti hisoblanadi:
```bash
passwd -l root
exit
```

---

### 4.2. Xost Kompyuterdan SSH Orqali Ulanish

Serverda SSH xizmati ishlayotganiga ishonch hosil qiling:
```bash
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

Xost kompyuteringizdan (Windows PowerShell, CMD yoki Linux/macOS terminali) serverning **NAT IP manzili** (yoki agar marshrut sozlangan bo'lsa Host-Only IP) orqali ulanish sintaksisi:

```bash
# Sintaksis: ssh <foydalanuvchi_nomi>@<SERVER_IP>
ssh adminlab@192.168.220.130
```

Agar maxsus port ishlatilsa:
```bash
ssh -p 22 adminlab@192.168.220.130
```

---

## 5. Byudjetli Noutbuklar Uchun Resurslarni Optimallashtirish (Disk, RAM, CPU)

Kam resursli (masalan: 8–16 GB RAM, 4–6 yadroli CPU, 256–512 GB SSD) noutbuklarda bir vaqtning o'zida 3–4 ta virtual mashinani qotib qolmasdan (freeze bo'lmasdan) ishlatish uchun quyidagi qat'iy optimallashtirish qoidalariga amal qiling.

---

### 5.1. Disk Joyini Tejash: Yagona ISO va Uni O'z Vaqtida Uzish (Disconnect)

* **Xato amaliyot:** Har bir VM uchun alohida `ubuntu-server.iso` nusxasini yuklab olish yoki turli papkalarga ko'chirib saqlash (har bir ISO ~2.5 GB joy oladi).
* **To'g'ri amaliyot:**
  1. Xost kompyuterda bitta umumiy katalog yarating (masalan: `D:\ISO\` yoki `C:\VMs\ISOs\`).
  2. Barcha 3–4 ta virtual mashinaning CD/DVD drayverini aynan bitta shu `.iso` faylga ulang.
  3. **O'rnatish tugagach ISO ni uzish:** Tizim o'rnatilib bo'lingach, ISO fayli xotirada qolmasligi va VM ishga tushishida disk resurslarini yuklamasligi uchun CD/DVD drayverini uzib qo'ying:
     - **VM Settings -> CD/DVD (SATA/IDE)** bo'limiga kiring.
     - **Device status** qismidagi **"Connected"** va **"Connect at power on"** belgilarini (checkbox) olib tashlang.
     - Connection bo'limida **"Use physical drive"** ga o'tkazib qo'ying yoki ISO yo'lini bo'shating.

---

### 5.2. Dynamic (Thin) vs Pre-allocated (Thick) Disk Tanlovi

VMware da virtual disk yaratishda ikkita muhim parametr mavjud:

| Parametr | Tavsif | Byudjetli noutbuk uchun tavsiya |
| :--- | :--- | :--- |
| **Allocate all disk space now** (Pre-allocated / Thick) | Belgilangan hajm (masalan, 25 GB) diskdan darhol to'liq kesib olinadi. 4 ta VM = 100 GB SSD! | **ASLO YOQMANG (Tavsiya etilmaydi)** |
| **Dynamic Provisioning** (Thin Provisioning) | Disk faqat ichiga real fayllar yozilishi bilan asta-sekin kengayadi (boshida 2.5–3 GB joy oladi). | **QAT'IY TAVSIYA ETILADI** |

> [!TIP]
> Disk yaratishda **"Split virtual disk into multiple files"** opsiyasini tanlash tavsiya etiladi. Bu fayllarni xost kompyuterda defragmentatsiya qilish, boshqa joyga ko'chirish va zaxiralashni ancha osonlashtiradi.

---

### 5.3. Linked Clone vs Full Clone: Disk Hajmini 80% Tejash

Laboratoriya muhitida bir xil operatsion tizimli 3–4 ta server kerak bo'lganda **Linked Clone** eng samarali yechim hisoblanadi:

```
  +-------------------------------------------------------------+
  |              Asosiy VM ("Golden Master Image")              |
  |             (Toza o'rnatilgan Ubuntu Server)                |
  +-------------------------------------------------------------+
                                |
                 [Snapshot: "Clean-Base-State"]
                                |
         +----------------------+----------------------+
         |                                             |
+------------------+                          +------------------+
|  Linked Clone 1  |                          |  Linked Clone 2  |
|      (srv1)      |                          |      (srv2)      |
|  Delta: ~500 MB  |                          |  Delta: ~500 MB  |
+------------------+                          +------------------+
```

* **Full Clone (To'liq klon):** Boshlang'ich VM dan to'liq mustaqil nusxa ko'chiradi. Har bir klon yana 5–10 GB joy oladi.
* **Linked Clone (Bog'langan klon):** Asosiy VM ning ma'lum bir **Snapshot** (holat surati)iga tayanadi. Yangi klon faqat o'zgargan fayllarni (delta) yozadi. Boshlanishida har bir yangi VM atigi **300–500 MB** disk joyini band qiladi!

#### "Golden Image" va Snapshot Qoidalari:
1. Bitta "Master" VM o'rnating, barcha kerakli bazaviy paketlarni (`curl`, `git`, `vim`, `net-tools`) o'rnating.
2. Mashinani to'liq o'chiring (**Shut down**).
3. **VM -> Snapshot -> Take Snapshot...** ni tanlang (Nomi: `Golden-Base-Image`).
4. Klon yaratish: **VM -> Manage -> Clone... -> An existing snapshot -> Create a linked clone**.
5. **QAT'IY QOIDA:** Asosiy "Master" VM ni yoki uning ushbu Snapshotini **hech qachon o'chirib yubormang va joyini ko'chirmang**! Agar asosiy disk o'chsa, barcha bog'langan klonlar yaroqsiz holga keladi.

---

### 5.4. RAM va vCPU Taqsimoti: Tizim Qotishini (Freeze) Oldini Olish

Byudjetli noutbukda xost operatsion tizim (Windows/Linux) o'zining barqaror ishlashi uchun kamida **4 GB RAM** va **2 ta fizik yadro** talab qiladi.

#### Resurslarni taqsimlash formulasi:
Agar noutbukingizda **16 GB RAM** va **4 yadro / 8 potok (vCPU)** bo'lsa:
* Xost tizim uchun zaxira: **4–6 GB RAM**
* 4 ta VM uchun jami mavjud: **10–12 GB RAM**
* **Har bir VM ga tavsiya etilgan resurs:**
  - **RAM:** `1.5 GB` – `2 GB` (Ubuntu Server GUI siz, ya'ni Headless holatda ishlaganda 1 GB ham bemalol yetadi).
  - **vCPU:** `1 ta` yoki maksimal `2 ta` vCPU. Aslo noutbukdagi barcha jami yadrolar sonini bitta VM ga ajratmang!

#### Ortiqcha Virtual Uskunalarni O'chirish:
Server rejimida grafik interfeys kerak emas. Keraksiz virtual qurilmalar xostning RAM va CPU davrlarini behuda sarflaydi:
1. **VM Settings -> Display:** **"Accelerate 3D graphics"** belgisini olib tashlang (o'chiring). Bu video xotirani va GPU yuklamasini keskin kamaytiradi.
2. **VM Settings -> Sound Card:** O'chirib tashlang (**Remove**).
3. **VM Settings -> Printer:** O'chirib tashlang (**Remove**).
4. **VM Settings -> USB Controller:** Agar laboratoriyada fleshka ulanmasa, o'chirib qo'ying.

---

## 6. Laboratoriya Serverlari Uchun Tezkor Tekshiruv Cheklisti

Yangi server klonlangach, laboratoriyani ishga tushirishdan oldin 3 daqiqalik tekshiruv amallari:

| # | Tekshiruv Bosqichi | Buyruq / Harakat | Kutilayotgan Natija |
|---|---|---|---|
| 1 | **MAC Manzil** | `VM Settings -> Network Adapter -> Advanced` | Har bir VM da MAC unikal va `Latency = 0` |
| 2 | **Machine ID** | `cat /etc/machine-id` | Boshqa VM lar bilan bir xil bo'lmagan yangi 32 belgili hex-kod |
| 3 | **Hostname** | `hostname` | Belgilangan unikal server nomi (masalan: `srv1`) |
| 4 | **Hosts fayli** | `grep 127.0.1.1 /etc/hosts` | `127.0.1.1 <yangi_hostname>` |
| 5 | **Tarmoq kartalari** | `ip -br addr` | `ens33` (Statik IP), `ens37` (NAT IP) faol (`UP`) |
| 6 | **Ichki aloqa** | `ping -c 2 192.168.57.1` | Paket yo'qolishisiz muvaffaqiyatli javob |
| 7 | **Tashqi internet** | `ping -c 2 google.com` | Tashqi DNS va internet aloqasi mavjud |
| 8 | **Foydalanuvchi** | `whoami && id` | Yangi login nomi va `sudo` guruh a'zoligi |
| 9 | **CD/DVD holati** | `VM Settings -> CD/DVD` | `Connected at power on` o'chirilgan (unmounted) |

---
*Ushbu qo'llanma tarmoq xavfsizligi va tizim ma'murligi laboratoriyalarida mustahkam va barqaror infratuzilma yaratish uchun standart ko'rsatma hisoblanadi.*
