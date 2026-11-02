# A2. TLS, SSH va IPsec Protokollarini Sozlash
## Ikki Ubuntu Server O‘rtasida Ko‘p Qatlamli Himoyalangan Aloqa Kanalini Yaratish
### To‘liq Amaliy Laboratoriya Qo‘llanmasi va Tahliliy Hisobot

---

> **MUHIM ESLATMA (MUHIT VA TARMOQ BO‘YICHA):**
> * **Host operatsion tizim:** Ushbu qo‘llanma **Ubuntu Linux** asosiy tizimida (Host OS) bajarishga to‘liq moslashtirilgan. Windows muhiti ishlatilmaydi!
> * **Virtualizatsiya:** Ubuntu Host ichida **VMware Workstation Pro / Player for Linux** o‘rnatiladi va uning ichida 2 ta **Ubuntu Server 24.04 LTS** virtual mashinasi (`CLIENT-1` va `SRV-1`) ishga tushiriladi.
> * **Lokal Wi-Fi tarmog‘i:** Foydalanuvchi universitet Wi-Fi tarmog‘ida emas, o‘zining **shaxsiy lokal Wi-Fi** tarmog‘ida ishlaydi. Shu sababli:
>   - Virtual mashinalar paketlarni o‘rnatishda hostning lokal Wi-Fi adapteriga ulangan vaqtinchalik **NAT** orqali internetga chiqadi.
>   - Amaliyot va xavfsizlik sinovlari esa tashqi Wi-Fi tarmog‘iga mutlaqo daxlsiz bo‘lgan, maxsus ajratilgan izolyatsiyalangan virtual segmentda (**A2-LAB**, `192.168.57.0/24`) olib boriladi.
>   - Foydalanuvchining shaxsiy Wi-Fi tarmog‘iga laboratoriya trafigi sizib chiqmaydi, lokal router sozlamalariga hech qanday o‘zgartirish kiritish talab etilmaydi.
> * **CLI Buyruqlar:** Har bir bajariladigan buyruq (command) va uning har bir parametri, flagi nima vazifa bajarishi qat’iy va batafsil tushuntirib berilgan.

---

## MUNDARIJA
1. [1-BO‘LIM: Mavzuning Maqsadi va Xavfsizlik Konsepsiyasi](#1-bolim-mavzuning-maqsadi-va-xavfsizlik-konsepsiyasi)
   - [1.1. Laboratoriya ishining asosiy maqsadi](#11-laboratoriya-ishining-asosiy-maqsadi)
   - [1.2. Kutiladigan amaliy natijalar](#12-kutiladigan-amaliy-natijalar)
   - [1.3. Ko‘p qatlamli himoya (Defense-in-Depth) va protokollar solishtirmasi](#13-kop-qatlamli-himoya-defense-in-depth-va-protokollar-solishtirmasi)
   - [1.4. Laboratoriya xavfsizlik chegaralari va tahdid modeli](#14-laboratoriya-xavfsizlik-chegaralari-va-tahdid-modeli)
2. [2-BO‘LIM: Nazariy Asoslar va Arxitektura Qo‘llanmasi](#2-bolim-nazariy-asoslar-va-arxitektura-qollanmasi)
   - [2.1. Bajarish muhiti: Ubuntu Host + VMware Workstation for Linux](#21-bajarish-muhiti-ubuntu-host--vmware-workstation-for-linux)
   - [2.2. Lokal Wi-Fi sharoitida xavfsiz izolyatsiyalangan tarmoq arxitekturasi](#22-lokal-wi-fi-sharoitida-xavfsiz-izolyatsiyalangan-tarmoq-arxitekturasi)
   - [2.3. SSH (Secure Shell) va ED25519 kalitli autentifikatsiya, Hardening](#23-ssh-secure-shell-va-ed25519-kalitli-autentifikatsiya-hardening)
   - [2.4. TLS (Transport Layer Security), X.509 PKI, Lokal CA va SAN](#24-tls-transport-layer-security-x509-pki-lokal-ca-va-san)
   - [2.5. Nginx HTTPS veb-serveri va TLS 1.2/1.3 xususiyatlari](#25-nginx-https-veb-serveri-va-tls-1213-xususiyatlari)
   - [2.6. IPsec (IKEv2, ESP) va Linux yadrosi XFRM quyi tizimi](#26-ipsec-ikev2-esp-va-linux-yadrosi-xfrm-quyi-tizimi)
   - [2.7. Nazoratli salbiy (Controlled Negative) sinovlarning ahamiyati](#27-nazoratli-salbiy-controlled-negative-sinovlarning-ahamiyati)
3. [3-BO‘LIM: Mashqni Bajarish Ketma-ketligi (To‘liq Bosqichma-bosqich Amaliy Qo‘llanma)](#3-bolim-mashqni-bajarish-ketma-ketligi-toliq-bosqichma-bosqich-amaliy-qollanma)
   - [0-bosqich: Ubuntu Hostda VMware va VMlarni sozlash, paketlarni o‘rnatish](#0-bosqich-ubuntu-hostda-vmware-va-vmlarni-sozlash-paketlarni-ornatish)
   - [1-bosqich: Virtual tarmoq va IP manzillarni sozlash (A2-LAB: 192.168.57.0/24)](#1-bosqich-virtual-tarmoq-va-ip-manzillarni-sozlash-a2-lab-19216857024)
   - [2-bosqich: SSH ni ED25519 kaliti bilan sozlash va xavfsizlashtirish (Hardening)](#2-bosqich-ssh-ni-ed25519-kaliti-bilan-sozlash-va-xavfsizlashtirish-hardening)
   - [3-bosqich: Lokal CA va TLS server sertifikatini yaratish (SAN bilan)](#3-bosqich-lokal-ca-va-tls-server-sertifikatini-yaratish-san-bilan)
   - [4-bosqich: Nginx da TLS 1.2/1.3 HTTPS xizmatini ishga tushirish va sinovlar](#4-bosqich-nginx-da-tls-1213-https-xizmatini-ishga-tushirish-va-sinovlar)
   - [5-bosqich: strongSwan orqali IKEv2/IPsec transport rejimini sozlash](#5-bosqich-strongswan-orqali-ikev2ipsec-transport-rejimini-sozlash)
   - [6-bosqich: IPsec SA, Linux XFRM siyosati va tcpdump orqali shifrlangan trafik tahlili](#6-bosqich-ipsec-sa-linux-xfrm-siyosati-va-tcpdump-orqali-shifrlangan-trafik-tahlili)
   - [7-bosqich: Nazoratli xatolik sinovi: Noto‘g‘ri PSK va tizimni tiklash](#7-bosqich-nazoratli-xatolik-sinovi-notogri-psk-va-tizimni-tiklash)
   - [8-bosqich: Yakuniy integratsion tekshiruv va laboratoriyani xavfsiz yakunlash](#8-bosqich-yakuniy-integratsion-tekshiruv-va-laboratoriyani-xavfsiz-yakunlash)
   - [9-bosqich: Natija jadvallari va o‘qituvchi himoya savollariga to‘liq javoblar](#9-bosqich-natija-jadvallari-va-oqituvchi-himoya-savollariga-toliq-javoblar)
4. [4-BO‘LIM: Foydalanilgan Rasmiy Manbalar va Standartlar](#4-bolim-foydalanilgan-rasmiy-manbalar-va-standartlar)

---

## 1-BO‘LIM: Mavzuning Maqsadi va Xavfsizlik Konsepsiyasi

### 1.1. Laboratoriya ishining asosiy maqsadi
Ushbu laboratoriya ishining bosh maqsadi — kompyuter tarmoqlarida axborot xavfsizligini ta’minlashning eng muhim ustunlari bo‘lgan **SSH (Secure Shell)**, **TLS (Transport Layer Security)** va **IPsec (Internet Protocol Security)** kriptografik protokollarini real Linux muhitida amaliy sozlash, ularning o‘zaro ishlash mexanizmini va himoya qamrovini tadqiq qilishdan iborat.

Amaliyot davomida talaba:
1. **OpenSSH** xizmatini parolsiz, zamonaviy elliptik egri chiziqqa asoslangan **ED25519** asimmetrik kalitlari asosida sozlaydi va server xavfsizligini kuchaytiradi (hardening).
2. O‘zining shaxsiy **Lokal Sertifikatlash Markazini (Local CA)** quradi, X.509 v3 standartidagi zamonaviy **SAN (Subject Alternative Name)** maydoniga ega raqamli sertifikat ishlab chiqadi va **Nginx** veb-serverida faqat xavfsiz **TLS 1.2 / TLS 1.3** protokollarini yoqadi.
3. Linux yadrosining **XFRM** moduli va **strongSwan** dasturiy ta’minoti yordamida ikkita host o‘rtasida **IKEv2** va **ESP (Encapsulating Security Payload)** transport rejimiga asoslangan **IPsec** tunnelini o‘rnatadi.
4. Har bir protokolning to‘g‘ri ishlashini **ijobiy (positive)** sinovlar bilan tasdiqlaydi hamda ataylab kiritilgan xatoliklar orqali **nazoratli salbiy (controlled negative)** sinovlar o‘tkazib, xavfsizlik mexanizmlarining rad etish qobiliyatini tekshiradi.

---

### 1.2. Kutiladigan amaliy natijalar
Laboratoriya ishini muvaffaqiyatli yakunlagan talaba quyidagi ko‘nikma va natijalarga ega bo‘ladi:
* **Qatlamlar farqini tushuntirish:** SSH, TLS va IPsec protokollari TCP/IP va OSI modelining qaysi qatlamida ishlashi, qanday trafikni (terminal, veb-ilova yoki butun IP yuklama) himoyalashini tahlil qiladi.
* **Ishonch zanjiri (Chain of Trust) tahlili:** Raqamli sertifikatlarda `Subject`, `Issuer` va `Subject Alternative Name (SAN)` maydonlarining rolini tushunadi; nima uchun an’anaviy `CN (Common Name)` o‘rniga SAN ishlatilishi shartligini amalda ko‘rsatadi.
* **SSH Hardening amaliyoti:** Parol orqali kirishni butunlay o‘chirib, parollarni buzish (brute-force) va lug‘at hujumlaridan serverni 100% himoya qiladi.
* **IPsec va Yadro tahlili:** Linux yadrosining `ip xfrm state` (SAD) va `ip xfrm policy` (SPD) quyi tizimini o‘qiydi, `ipsec statusall` orqali IKE SA va CHILD SA holatlarini izohlaydi.
* **Paketlar darajasidagi dalil:** `tcpdump` yordamida shifrlanmagan ICMP paketi bilan IPsec ostidagi ESP (protokol 50) paketining tuzilishini solishtirib beradi.
* **Nosozliklarni bartaraf etish (Rollback):** Noto‘g‘ri kiritilgan konfiguratsiyani (masalan, noto‘g‘ri PSK yoki buzilgan SSH sozlamasini) aniqlab, xizmatni uzmasdan zaxiradan tiklay oladi.

---

### 1.3. Ko‘p qatlamli himoya (Defense-in-Depth) va protokollar solishtirmasi

Ko‘p qatlamli xavfsizlik (Defense-in-Depth) tamoyiliga ko‘ra, tarmoqda birgina xavfsizlik mexanizmiga tayanib qolish xavflidir. Agar bir qatlam buzilsa yoki chetlab o‘tilsa, keyingi qatlam ma’lumotlar maxfiyligi va yaxlitligini saqlab qolishi lozim.

Ushbu laboratoriyada 3 ta protokol bir vaqtning o‘zida ishlatiladi. Biroq bu "har doim barcha ma’lumotni uch marta ketma-ket shifrlash kerak" degani emas. Maqsad — turli qatlamlardagi protokollarning vazifasini taqqoslashdir:

| Xususiyat / Protokol | SSH (Secure Shell) | TLS (Transport Layer Security) | IPsec (Transport Rejimi) |
| :--- | :--- | :--- | :--- |
| **OSI Qatlami** | Ilova / Amaliy qatlam (Application Layer - L7) | Transport va Seans oralig‘i (L4–L7) | Tarmoq qatlami (Network Layer - L3) |
| **Himoyalanadigan oqim** | Masofaviy boshqaruv terminali, SCP, SFTP (TCP 22) | Veb-ilova sessiyasi, HTTPS so‘rovlar (TCP 443) | Ikki host orasidagi barcha IP paket yuklamasi (L4 payload) |
| **Autentifikatsiya usuli** | Server host kaliti + Mijozning ED25519 asimmetrik kaliti | Lokal CA tomonidan imzolangan X.509 server sertifikati | IKEv2 protokoli va Pre-Shared Key (PSK) siri |
| **Ishonch manbai** | `known_hosts` (TOFU) va serverdagi `authorized_keys` | Mijoz ishonadigan ildiz sertifikat (`ca.crt`) | Ikkala tomonda o‘rnatilgan maxfiy simmetrik kalit (PSK) |
| **Qamrov sohasi** | Faqat SSH dasturi sessiyasini shifrlaydi | Faqat TLS ni qo‘llab-quvvatlovchi dastur sessiyasini shifrlaydi | OS yadrosi darajasida ishlaydi, ilovalarga shaffof (transparent) |
| **Asosiy tekshiruv buyruqlari** | `ssh -v`, `sshd -T` | `curl --cacert`, `openssl s_client` | `ipsec statusall`, `ip xfrm state`, `tcpdump` |

---

### 1.4. Laboratoriya xavfsizlik chegaralari va tahdid modeli

1. **Xavfsizlik chegarasi (Security Boundary):**
   * Laboratoriyadagi barcha amallar faqat ikkita virtual mashina (`192.168.57.10` va `192.168.57.20`) o‘rtasida o‘rnatilgan izolyatsiyalangan **A2-LAB** virtual tarmog‘ida bajariladi.
   * Hostning tashqi Wi-Fi adapteriga yoki tashqi tarmoqlarga hech qanday test paketlari, sintetik xatolar yoki zaif parollar yuborilmaydi.
2. **Tahdid modeli va cheklovlar:**
   * **Maxfiy kalitlar:** Shaxsiy kalitlar (`ca.key`, `srv1.key`, `~/.ssh/a2_ed25519`) va laboratoriya PSK siri hech qachon hisobotlarga, ochiq skrinshotlarga yoki umumiy papkalarga joylanmasligi shart.
   * **PSK laboratoriya sharti:** Laboratoriyada qulaylik uchun PSK tanlangan. Haqiqiy yirik korporativ ishlab chiqarish (production) muhitida IKEv2 uchun o‘zaro raqamli sertifikatlar (X.509) orqali autentifikatsiya qilish talab etiladi.

---

## 2-BO‘LIM: Nazariy Asoslar va Arxitektura Qo‘llanmasi

### 2.1. Bajarish muhiti: Ubuntu Host + VMware Workstation for Linux

Sizning kompyuteringizda asosiy operatsion tizim sifatida **Ubuntu Linux (Desktop)** ishlayapti. Unda virtualizatsiya dasturi sifatida **VMware Workstation Pro yoki Player for Linux** o‘rnatiladi. 

Ushbu arxitekturaning afzalliklari:
1. **Yadro darajasidagi virtualizatsiya:** Linux hostda ishlayotgan VMware KVM yoki o‘zining `vmmon`/`vmnet` drayverlari orqali apparat tezlanishini (Intel VT-x / AMD-V) to‘g‘ridan-to‘g‘ri taqdim etadi.
2. **Virtual tarmoq boshqaruvi:** Linux ostidagi VMware `vmware-netcfg` (Virtual Network Editor) yoki `/etc/vmware/networking` konfiguratsiya fayli orqali virtual tarmoqlarni boshqaradi.
3. **VMlar:**
   * **CLIENT-1:** Ubuntu Server 24.04 LTS (2 vCPU, 2 GB RAM, 20 GB disk).
   * **SRV-1:** Ubuntu Server 24.04 LTS (2 vCPU, 2 GB RAM, 20 GB disk).

```
+-----------------------------------------------------------------------------------------+
|                              UBUNTU LINUX HOST TIZIMI                                    |
|                                                                                         |
|  [ Sizning Lokal Wi-Fi Tarmog'ingiz (wlan0 / wlp2s0) ] ---> Router ---> Internet         |
|                                                                                         |
|  +-----------------------------------------------------------------------------------+  |
|  |                    VMware Workstation Pro / Player for Linux                      |  |
|  |                                                                                   |  |
|  |   +-----------------------+                         +-----------------------+     |  |
|  |   |    VM 1: CLIENT-1     |                         |     VM 2: SRV-1       |     |  |
|  |   | (Ubuntu Server 24.04) |                         | (Ubuntu Server 24.04) |     |  |
|  |   |                       |                         |                       |     |  |
|  |   | [Adapter 1: NAT] ---->|--+ (Vaqtinchalik apt) +-->| [Adapter 1: NAT]      |     |  |
|  |   | (Lokal Wi-Fi orqali)  |  |                     |  | (Lokal Wi-Fi orqali)  |     |  |
|  |   |                       |  |                     |  |                       |     |  |
|  |   | [Adapter 2: VMnet2]   |  |                     |  | [Adapter 2: VMnet2]   |     |  |
|  |   | 192.168.57.10/24      |  |                     |  | 192.168.57.20/24      |     |  |
|  |   +-----------|-----------+  |                     +-----------|-----------+     |  |
|  +---------------|--------------|---------------------------------|------------------+  |
|                  |              v (O'rnatishdan keyin o'chiriladi)|                     |
|                  +==============+=================================+                     |
|                                 |                                                       |
|              [ Virtual Tarmoq: A2-LAB (Custom Host-Only / vmnet2) ]                     |
|                                 Subnet: 192.168.57.0/24                                 |
|                    (DHCP yo'q, Gateway yo'q, DNS yo'q, To'liq Izolyatsiya)              |
+-----------------------------------------------------------------------------------------+
```

---

### 2.2. Lokal Wi-Fi sharoitida xavfsiz izolyatsiyalangan tarmoq arxitekturasi

Universitet Wi-Fi tarmog‘idan farqli ravishda, siz o‘zingizning uyingizdagi yoki shaxsiy lokal Wi-Fi tarmog‘ingizdasiz. Lokal Wi-Fi routerlari odatda `192.168.1.0/24`, `192.168.0.0/24` yoki `10.0.0.0/24` diapazonida ishlaydi.

Laboratoriya talabi bo‘yicha tarmoq segmenti: **192.168.57.0/24** bo‘lishi shart!

Xavfsiz va to‘g‘ri ishlash qoidalari:
1. **Ikki adapterli yondashuv (Dual-NIC strategy):**
   * **Adapter 1 (NAT):** Ubuntu Serverlarga kerakli dasturlarni (`strongswan`, `nginx`, `tcpdump` va h.k.) yuklab olish uchun ishlatiladi. VMware NAT mexanizmi hostning Wi-Fi interfeysiga ulanadi va virtual mashinalarga internet beradi.
   * **Adapter 2 (Custom / Host-Only — Masalan: `vmnet2`):** Bu A2-LAB segmenti bo‘ladi. VMware da ushbu tarmoqda **DHCP serveri o‘chirib qo‘yiladi**. Unda Gateway va DNS bo‘lmaydi.
2. **Izolyatsiya kafolati:** Dasturlar o‘rnatilgach, laboratoriya boshlanishidan oldin Adapter 1 (NAT) interfeysi tizimda vaqtincha to‘xtatiladi (`ip link set down`). Barcha amaliyot faqat `192.168.57.0/24` ichida kechadi. Natijada:
   * Sizning shaxsiy Wi-Fi tarmog‘ingizdagi boshqa qurilmalar (telefon, televizor, oila a’zolari noutbuki) laboratoriya paketlarini ko‘rmaydi.
   * Laboratoriyadagi IPsec yoki SSH sinovlari shaxsiy Wi-Fi routeringizning xavfsizlik devori (firewall) tomonidan bloklanmaydi yoki routerga yuk tushirmaydi.

---

### 2.3. SSH (Secure Shell) va ED25519 kalitli autentifikatsiya, Hardening

SSH protokoli (TCP port 22) masofaviy boshqaruvni ta’minlovchi kriptografik standartdir.

1. **Nega ED25519?**
   * An’anaviy RSA (2048 yoki 4096 bit) kalitlari katta hajmdagi matematik hisoblashlarni talab qiladi.
   * **ED25519** — EdDSA (Edwards-curve Digital Signature Algorithm) va Curve25519 elliptik egri chizig‘iga asoslangan. U atigi 256 bit hajmga ega bo‘lsa-da, 3072 bitli RSA ga teng yoki undan yuqori xavfsizlik darajasini taqdim etadi, hisoblash tezligi ancha yuqori va yon kanallar (side-channel) hujumlariga chidamli.
2. **KDF Rounds (`-a 64`):**
   * Maxfiy kalitni diskda shifrlab saqlashda bcrypt-PBKDF algoritmi qo‘llaniladi. 64 raund kiritilishi xakerlarning maxfiy kalit parolini GPU yordamida tezkor brute-force qilishini keskin qiyinlashtiradi.
3. **SSH Hardening mohiyati:**
   * `PermitRootLogin no`: Tizim administratori hisobiga to‘g‘ridan-to‘g‘ri kirishni taqiqlaydi.
   * `PasswordAuthentication no` va `KbdInteractiveAuthentication no`: Parolli autentifikatsiyani butunlay o‘chiradi. Faqat ro‘yxatdan o‘tgan ochiq kalit egalarigina kira oladi.
   * `MaxAuthTries 3`: Bir ulanishda ruxsat berilgan urinishlar sonini cheklaydi.
   * `AllowUsers student`: Faqat ruxsat etilgan foydalanuvchigina ulanishi mumkin.

---

### 2.4. TLS (Transport Layer Security), X.509 PKI, Lokal CA va SAN

TLS protokoli ilova sathi protokollarini (masalan, HTTP -> HTTPS) xavfsiz qilish uchun xizmat qiladi.

1. **Ochiq Kalitlar Infratuzilmasi (PKI) va Lokal CA:**
   * Ommaviy internetda sertifikatlarni Let's Encrypt, DigiCert kabi global CA lar imzolaydi.
   * Laboratoriya va ichki korporativ tarmoqlarda tashqi internetga bog‘lanmagan **Lokal CA** yaratiladi.
   * CA ning o‘ziga xos xususiyati: o‘zini o‘zi imzolagan ildiz sertifikat (`ca.crt`) va eng muhim maxfiy kalit (`ca.key`).
2. **SAN (Subject Alternative Name) — Nima uchun CN eskirgan?**
   * Ilgari domen nomi faqat sertifikatning `Common Name (CN)` maydoniga yozilar edi.
   * RFC 2818 va zamonaviy xavfsizlik talablariga (Chrome, Firefox, curl) ko‘ra, CN maydoni eskirgan deb topilgan. Mijozlar sertifikatni tekshirishda faqat **SAN (X.509 v3 extension)** maydoniga qaraydi.
   * SAN bir vaqtning o‘zida ham domen nomini (`DNS:srv1.a2.lab`), ham to‘g‘ridan-to‘g‘ri IP manzilni (`IP:192.168.57.20`) ko‘rsatish imkonini beradi.
3. **Ishonch zanjiri (Chain of Trust):**
   * Mijoz (`CLIENT-1`) serverga ulanayotganda server unga `srv1.crt` ni beradi.
   * Mijoz bu sertifikat kim tomonidan imzolanganini tekshiradi (`Issuer: CN=A2 Lab CA`).
   * Agar mijozning ishonchli sertifikatlar ro‘yxatida ushbu CA mavjud bo‘lsa va sertifikat muddati, SAN mos kelsa, aloqa xavfsiz deb tan olinadi (`Verify return code: 0 (ok)`).

---

### 2.5. Nginx HTTPS veb-serveri va TLS 1.2/1.3 xususiyatlari

* **TLS 1.0 va 1.1:** Eskirgan, zaif shifrlash usullariga ega (CBC rejimidagi zaifliklar, POODLE, BEAST hujumlari) va zamonaviy tarmoqlarda taqiqlangan.
* **TLS 1.2 va TLS 1.3:** Faqat kuchli AEAD (Authenticated Encryption with Associated Data) shifrlash algoritmlarini (AES-GCM, CHACHA20-POLY1305) qo‘llaydi. TLS 1.3 handshakeni 1-RTT gacha qisqartirib, ulanish tezligini oshiradi va to‘liq Forward Secrecy (PFS) ta’minlaydi.
* Nginx konfiguratsiyasida `ssl_protocols TLSv1.2 TLSv1.3;` direktivasi orqali eski va xavfli protokollar qat’iyan man etiladi.

---

### 2.6. IPsec (IKEv2, ESP) va Linux yadrosi XFRM quyi tizimi

IPsec — tarmoq sathi (L3) xavfsizlik protokolidir. U ilovalardan mutlaqo mustaqil ishlaydi.

1. **Transport Rejimi vs Tunnel Rejimi:**
   * **Tunnel Rejimi:** Butun IP paketi (eski IP header + payload) yangi IP header ichiga o‘raladi. Odatda Gateway-to-Gateway (saytlararo VPN) uchun qo‘llaniladi.
   * **Transport Rejimi:** Asl IP sarlavhasi (IP header: `192.168.57.10 -> 192.168.57.20`) ochiq saqlanadi, faqat uning ichidagi foydali yuklama (TCP/UDP/ICMP ma’lumotlari) **ESP** yordamida shifrlanadi. Bu Host-to-Host ulanishlar uchun ideal va sarlavha qo‘shimcha yukini kamaytiradi.
2. **IKEv2 (Internet Key Exchange v2):**
   * UDP 500 va 4500 portlarida ishlaydi.
   * Vazifasi: Ikki tomon o‘rtasida simmetrik shifrlash kalitlarini xavfsiz kelishib olish (Diffie-Hellman), tomonlarni autentifikatsiya qilish va SA (Security Association) parametrlarini o‘rnatish.
3. **ESP (Encapsulating Security Payload - IP Protokol 50):**
   * Kelishilgan kalitlar asosida ma’lumotlarni AES-256 bilan shifrlaydi va SHA-256 bilan yaxlitligini kafolatlaydi.
4. **Linux Yadrosining XFRM (Transform) Quyi Tizimi:**
   * strongSwan faqat IKEv2 xizmati (control plane) hisoblanadi. U kalitlarni kelishib olgach, bu kalitlar va siyosatlarni to‘g‘ridan-to‘g‘ri Linux yadrosiga yuklaydi (data plane).
   * **SAD (Security Association Database):** `ip xfrm state` — qaysi SPI (Security Parameter Index), qaysi shifrlash kaliti va qaysi algoritm bilan paketlarni shifrlash/ochish kerakligini saqlaydi.
   * **SPD (Security Policy Database):** `ip xfrm policy` — qaysi IP diapazonlar orasidagi trafik IPsec orqali o‘tishi shartligini belgilaydi.

---

### 2.7. Nazoratli salbiy (Controlled Negative) sinovlarning ahamiyati

Axborot xavfsizligida faqat tizimning "ishlayotganini" ko‘rsatish (ijobiy test) yetarli emas. Eng asosiysi — tizim ruxsat berilmagan, soxtalashtirilgan yoki buzilgan urinishlarni **to‘g‘ri rad eta olishini** isbotlashdir:
1. **SSH:** Kalitsiz, faqat parol bilan kirishga uringanda server `Permission denied (publickey)` javobini berishi — parol orqali kirish bloklanganining yagona qat’iy isbotidir.
2. **TLS:** Sertifikatda mavjud bo‘lmagan soxta nom (`wrong.a2.lab`) bilan murojaat qilinganda `curl` ning xato berishi — mijoz dasturi Man-in-the-Middle (MITM) hujumlaridan himoyalanganini isbotlaydi.
3. **IPsec:** Noto‘g‘ri PSK kiritilganda ulanish o‘rnatilmasdan `AUTHENTICATION_FAILED` xatosi chiqishi — begona shaxs tarmoqqa noqonuniy ulana olmasligini ko‘rsatadi.

---

## 3-BO‘LIM: Mashqni Bajarish Ketma-ketligi (To‘liq Bosqichma-bosqich Amaliy Qo‘llanma)

> **ESLATMA:** Quyidagi barcha CLI buyruqlar yonida ularning vazifasi, har bir kalit so‘zi va parametri alohida tushuntirilgan. Buyruqlarni ketma-ket bajaring.

---

### 0-bosqich: Ubuntu Hostda VMware va VMlarni sozlash, paketlarni o‘rnatish

#### 0.1. VMware Workstation tarmoq sozlamasi (Ubuntu Hostda)
1. Ubuntu host tizimida terminalni oching va VMware Virtual Network Editor-ni ishga tushiring:
   ```bash
   sudo vmware-netcfg
   ```
2. Yangi virtual tarmoq qo‘shing (masalan, `vmnet2`):
   * Turi: **Host-only** (Connect VMs internally in a private network).
   * **Use local DHCP service to distribute IP address to VMs** belgisini **olib tashlang** (DHCP o‘chiriladi!).
   * **Subnet IP:** `192.168.57.0`, **Subnet Mask:** `255.255.255.0`.
   * Saqlang va chiqing.

#### 0.2. Virtual Mashinalarning adapter sozlamalari
Har ikkala VM (`CLIENT-1` va `SRV-1`) sozlamalariga kiring:
* **Network Adapter 1:** **NAT** (Hostning lokal Wi-Fi tarmog‘idan internet oladi — faqat paket o‘rnatish uchun).
* **Network Adapter 2:** **Custom (`/dev/vmnet2`)** (Izolyatsiyalangan `A2-LAB` tarmog‘i).

#### 0.3. Paketlarni o‘rnatish (Har ikkala VMda)
VMlarni yoqing. Hozircha ular Adapter 1 (NAT) orqali internetga chiqa oladi.

**CLIENT-1 da:**
```bash
sudo apt update
sudo apt install -y openssh-client openssl curl strongswan tcpdump iproute2
```
* `sudo apt update`: Paketlar ombori indekslarini yangilaydi.
* `openssh-client`: SSH orqali ulanish va `ssh-keygen`, `ssh-copy-id`, `scp` vositalarini taqdim etadi.
* `openssl`: Kriptografik operatsiyalar, sertifikat tekshirish uchun.
* `curl`: HTTP/HTTPS so‘rovlarini terminaldan yuborish vositasi.
* `strongswan`: IPsec IKEv2 demoni.
* `tcpdump`: Tarmoq paketlarini tahlil qilish uchun sniffer.
* `iproute2`: Zamonaviy Linux tarmoq boshqaruvi (`ip` buyrug‘i).

**SRV-1 da:**
```bash
sudo apt update
sudo apt install -y openssh-server openssl nginx strongswan tcpdump iproute2
sudo systemctl enable --now ssh nginx strongswan-starter
```
* `openssh-server`: SSH masofaviy kirish xizmati (`sshd`).
* `nginx`: Yuqori unumli veb-server (HTTPS uchun).
* `systemctl enable --now ssh nginx strongswan-starter`: Ushbu uchta servisni tizim yuklanganda avtomatik ishga tushadigan qiladi va hozirning o‘zidayoq faollashtiradi.

#### 0.4. Paketlar o‘rnatilgach, NAT adapterni uzish
Paketlar to‘liq o‘rnatilgach, ikkala VMda ham NAT adapterini o‘chirib qo‘ying (masalan, VMware oynasida Adapter 1 ni "Disconnect" qiling yoki VM ichida `sudo ip link set dev <NAT_IF> down` bajaring). Endi barcha ishlar sof **A2-LAB** segmentida kechadi.

---

### 1-bosqich: Virtual tarmoq va IP manzillarni sozlash (A2-LAB: 192.168.57.0/24)

#### 1.1. Interfeys nomini aniqlash va IP belgilash

Virtual tarmoq interfeysi nomini bilish uchun buyruq bajaramiz:
```bash
ip -br link
```
* `ip`: Linux yadro tarmoq boshqaruv vositasi.
* `-br` (`--brief`): Interfeyslar ro‘yxatini qisqa, ixcham jadval ko‘rinishida (Nom, Holat, MAC) chiqaradi.
* `link`: 2-qatlam (L2) tarmoq interfeyslari bilan ishlash bo‘limi.
* *Natijada `ens33`, `ens37` yoki `enp0s8` kabi nomlar ko‘rinadi. Quyidagi buyruqlarda ikkinchi (A2-LAB ga ulangan) interfeys nomini `LAB_IF` o‘zgaruvchisiga o‘zlashtiring.*

**CLIENT-1 da bajariladi:**
```bash
LAB_IF=ens37
sudo ip link set "$LAB_IF" up
sudo ip addr flush dev "$LAB_IF"
sudo ip addr add 192.168.57.10/24 dev "$LAB_IF"
ip -br addr show "$LAB_IF"
ip route get 192.168.57.20
```
*Komandalar tahlili:*
* `LAB_IF=ens37`: Interfeys nomini o‘zgaruvchiga yozish (agar sizda `ens38` yoki boshqa nom bo‘lsa, shuni yozing).
* `sudo ip link set "$LAB_IF" up`: Tanlangan tarmoq interfeysini faollashtiradi (link holatini UP qiladi).
* `sudo ip addr flush dev "$LAB_IF"`: Ushbu interfeysdagi barcha eski yoki dinamik olingan IP manzillarni tozalaydi.
* `sudo ip addr add 192.168.57.10/24 dev "$LAB_IF"`: Interfeysga statik `192.168.57.10` IP manzilini va `255.255.255.0` (`/24`) tarmoq niqobini biriktiradi.
* `ip -br addr show "$LAB_IF"`: Berilgan IP manzil interfeysda to‘g‘ri o‘tirganini tekshiradi.
* `ip route get 192.168.57.20`: Kernel ushbu manzilga paket jo‘natish uchun qaysi interfeysdan foydalanishini ko‘rsatadi.

**SRV-1 da bajariladi:**
```bash
LAB_IF=ens37
sudo ip link set "$LAB_IF" up
sudo ip addr flush dev "$LAB_IF"
sudo ip addr add 192.168.57.20/24 dev "$LAB_IF"
ip -br addr show "$LAB_IF"
sudo ss -lntup | grep -E ':(22|443)\b' || true
```
*Komandalar tahlili:*
* `sudo ip addr add 192.168.57.20/24 dev "$LAB_IF"`: Serverga statik `192.168.57.20` manzilini beradi.
* `sudo ss -lntup | grep -E ':(22|443)\b'`: Tizimda 22 (SSH) va 443 (Nginx) portlari tinglanayotganini tekshiradi:
  - `-l`: Faqat tinglanayotgan (listening) soketlarni ko‘rsatish.
  - `-n`: Port va IP larni nomga aylantirmasdan raqamda chiqarish.
  - `-t`: TCP soketlar.
  - `-u`: UDP soketlar.
  - `-p`: Qaysi jarayon (PID/Program name) ochganini ko‘rsatish.

#### 1.2. Bazaviy aloqani tekshirish
**CLIENT-1 da bajariladi:**
```bash
ping -c 3 192.168.57.20
ip neigh show 192.168.57.20
```
*Komandalar tahlili:*
* `ping -c 3 192.168.57.20`: SRV-1 serveriga 3 ta ICMP Echo Request paketi yuboradi (`-c 3` — 3 marta). Barcha 3 paket yo‘qotishsiz yetib borishi kerak.
* `ip neigh show 192.168.57.20`: ARP keshini tekshiradi. Bu L2 darajasida `192.168.57.20` manzilining MAC adresi muvaffaqiyatli aniqlanganini (`REACHABLE`) ko‘rsatadi.

---

### 2-bosqich: SSH ni ED25519 kaliti bilan sozlash va xavfsizlashtirish (Hardening)

#### 2.1. CLIENT-1 da ED25519 kalit juftligini yaratish va serverga ko‘chirish
CLIENT-1 terminalida:
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/a2_ed25519
```
*Komandalar tahlili:*
* `mkdir -p ~/.ssh`: Foydalanuvchining shaxsiy SSH katalogini yaratadi (mavjud bo‘lsa xato bermaydi).
* `chmod 700 ~/.ssh`: Katalog huquqlarini `drwx------` ga o‘rnatadi (faqat egasi o‘qiy oladi, bu SSH xavfsizlik talabidir).
* `ssh-keygen`: Yangi SSH asimmetrik kalit juftligini yaratuvchi buyruq.
* `-t ed25519`: Kalit turini zamonaviy Ed25519 elliptik egri chizig‘i qilib belgilaydi.
* `-a 64`: KDF (Key Derivation Function) uchun 64 ta raund qo‘llaydi (parol bilan shifrlanganda brute-force ga qarshi).
* `-f ~/.ssh/a2_ed25519`: Kalit fayli nomini belgilaydi. Natijada ikkita fayl hosil bo‘ladi: `a2_ed25519` (maxfiy kalit) va `a2_ed25519.pub` (ochiq kalit).

Endi ochiq kalitni SRV-1 serveridagi `student` foydalanuvchisiga o‘rnatamiz:
```bash
ssh-copy-id -i ~/.ssh/a2_ed25519.pub student@192.168.57.20
```
*Komanda tahlili:*
* `ssh-copy-id`: Ochiq kalitni masofaviy serverning `~/.ssh/authorized_keys` fayliga xavfsiz tarzda qo‘shib qo‘yuvchi utilita.
* `-i ~/.ssh/a2_ed25519.pub`: Qaysi ochiq kalit nusxalanishini ko‘rsatadi.
*(Eslatma: Bu vaqtda server hali parolni qabul qiladi, shuning uchun server foydalanuvchisi parolini kiritasiz).*

Kalit orqali dastlabki kirishni tekshirib ko‘ramiz:
```bash
ssh -i ~/.ssh/a2_ed25519 student@192.168.57.20 'hostname; whoami'
```
*Komanda tahlili:*
* `ssh`: Masofaviy serverga ulanish buyrug‘i.
* `-i ~/.ssh/a2_ed25519`: Autentifikatsiya uchun ko‘rsatilgan maxfiy kalit identifikatoridan foydalanish.
* `'hostname; whoami'`: Serverga kirib, uning nomini va foydalanuvchi kimligini chop etib, darhol chiqib ketish. Parol so‘ramasdan to‘g‘ridan-to‘g‘ri javob qaytishi kerak.

#### 2.2. SRV-1 da SSH Hardening (Xavfsizlikni kuchaytirish)
SRV-1 terminaliga o‘tamiz. Avval mavjud sozlamaning zaxira nusxasini olamiz:
```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.a2-backup
```
* `cp -a`: Faylni barcha egalik huquqlari, atributlari va vaqt tamg‘alari bilan arxiv rejimida nusxalaydi.

Endi `/etc/ssh/sshd_config.d/` katalogida yangi drop-in konfiguratsiya fayli yaratamiz:
```bash
sudo nano /etc/ssh/sshd_config.d/99-a2-hardening.conf
```
Fayl ichiga quyidagi qatorlarni yozing:
```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
AllowUsers student
```
*Konfiguratsiya tahlili:*
* `PermitRootLogin no`: `root` foydalanuvchisiga SSH orqali bevosita ulanishni butunlay taqiqlaydi.
* `PubkeyAuthentication yes`: Kriptografik ochiq kalitlar orqali ulanishga ruxsat beradi.
* `PasswordAuthentication no`: Standart parollar orqali ulanishni bekor qiladi.
* `KbdInteractiveAuthentication no`: Klaviatura orqali interaktiv parol kiritish so‘rovlarini o‘chiradi (PAM orqali parolni chetlab o‘tishning oldini oladi).
* `MaxAuthTries 3`: Bitta sessiyada xato urinishlar sonini maksimum 3 taga cheklaydi.
* `AllowUsers student`: Serverga faqatgina `student` hisobidan kirishga ruxsat beradi.

Sozlama sintaksisini tekshiramiz va SSH xizmatini qayta yuklaymiz:
```bash
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|maxauthtries|allowusers)'
sudo systemctl reload ssh
```
*Komandalar tahlili:*
* `sudo sshd -t`: SSH demonining konfiguratsiya sintaksisini tekshirish rejimi (test mode). Hech qanday xatolik chiqmasligi shart.
* `sudo sshd -T`: Hozir amalda bo‘lgan to‘liq konfiguratsiyani ekranga chiqaradi (kiritilgan direktivalar kuchga kirganini ko‘rsatadi).
* `sudo systemctl reload ssh`: Xizmatni to‘xtatmasdan, mavjud ulanishlarni uzmasdan konfiguratsiyani xavfsiz qayta yuklaydi.

#### 2.3. Ijobiy va nazoratli salbiy SSH sinovlari
CLIENT-1 terminaliga o‘tib, ikkita sinov o‘tkazamiz:

1. **Ijobiy sinov (Kalit bilan muvaffaqiyatli kirish):**
```bash
ssh -i ~/.ssh/a2_ed25519 student@192.168.57.20 'echo SSH_KEY_OK'
```
*Kutiladigan natija:* Server konsolida `SSH_KEY_OK` yozuvi chiqadi.

2. **Nazoratli salbiy sinov (Parol orqali kirish urinishi):**
```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password student@192.168.57.20
```
*Komanda parametrlari:*
* `-o PubkeyAuthentication=no`: Mijoz tomonidan kalitdan foydalanishni majburiy o‘chiradi.
* `-o PreferredAuthentications=password`: Serverdan faqat parol kiritish usulini talab qiladi.
* *Kutiladigan natija:* `student@192.168.57.20: Permission denied (publickey).` xabari chiqadi. Bu xato emas — bu server parollarni qabul qilmayotganining qat’iy isboti!

> **NOSOZLIKNI TIKLASH (ROLLBACK):**
> Agar biror xatolik tufayli SSH kalit bilan kirmay qolsa, SRV-1 konsolida:
> ```bash
> sudo rm -f /etc/ssh/sshd_config.d/99-a2-hardening.conf
> sudo cp -a /etc/ssh/sshd_config.a2-backup /etc/ssh/sshd_config
> sudo systemctl restart ssh
> ```
> buyruqlari orqali avvalgi holatga qaytariladi.

---

### 3-bosqich: Lokal CA va TLS server sertifikatini yaratish (SAN bilan)

Ushbu bosqich SRV-1 mashinasida bajariladi.

#### 3.1. Maxsus katalog va Ildiz CA (Certificate Authority) yaratish
```bash
sudo install -d -m 700 /etc/a2-tls
cd /etc/a2-tls
sudo openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out ca.key
sudo openssl req -x509 -new -sha256 -days 365 -key ca.key -out ca.crt -subj "/C=UZ/O=A2 Lab/CN=A2 Lab CA"
```
*Komandalar tahlili:*
* `sudo install -d -m 700 /etc/a2-tls`: `/etc/a2-tls` katalogini yaratadi (`-d`) va unga faqat `root` foydalanuvchisi kira oladigan `700` (`rwx------`) ruxsatini o‘rnatadi.
* `openssl genpkey`: Zamonaviy OpenSSL kalit generatori.
* `-algorithm RSA`: RSA algoritmini tanlaydi.
* `-pkeyopt rsa_keygen_bits:3072`: CA ning maxfiy kalit uzunligini 3072 bit qilib belgilaydi (yuqori xavfsizlik darajasi).
* `-out ca.key`: Yaratilgan maxfiy kalitni saqlash fayli.
* `openssl req -x509 -new`: Yangi o‘zini o‘zi imzolagan (self-signed) X.509 standartidagi ildiz sertifikat yaratish.
* `-sha256`: Sertifikat heshlash algoritmi sifatida SHA-256 ni belgilaydi.
* `-days 365`: Sertifikatning amal qilish muddati (1 yil).
* `-key ca.key`: Sertifikatni imzolash uchun boya yaratilgan CA maxfiy kalitidan foydalanadi.
* `-out ca.crt`: Tayyor ildiz sertifikat fayli.
* `-subj "/C=UZ/O=A2 Lab/CN=A2 Lab CA"`: Sertifikat egasi haqidagi ma’lumotlar (Country=UZ, Organization=A2 Lab, CommonName=A2 Lab CA).

#### 3.2. Server maxfiy kaliti va so‘rovi (CSR) yaratish
```bash
sudo openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out srv1.key
sudo openssl req -new -key srv1.key -out srv1.csr -subj "/C=UZ/O=A2 Lab/CN=srv1.a2.lab"
```
*Komandalar tahlili:*
* `openssl genpkey ... rsa_keygen_bits:2048 -out srv1.key`: Nginx serveri uchun 2048 bitli RSA maxfiy kalitini yaratadi.
* `openssl req -new -key srv1.key -out srv1.csr`: Server uchun sertifikat imzolash so‘rovini (CSR - Certificate Signing Request) yaratadi.
* `-subj "/C=UZ/O=A2 Lab/CN=srv1.a2.lab"`: Server nomi sifatida `srv1.a2.lab` ko‘rsatiladi.

#### 3.3. X.509 v3 SAN kengaytma faylini tayyorlash
Zamonaviy standartlar bo‘yicha sertifikat ichida SAN bo‘lishi shart. Fayl yaratamiz:
```bash
sudo nano /etc/a2-tls/srv1.ext
```
Fayl mazmuni:
```text
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=DNS:srv1.a2.lab,IP:192.168.57.20
```
*Kengaytmalar tahlili:*
* `basicConstraints=CA:FALSE`: Ushbu sertifikat boshqa sertifikatlarni imzolay olmasligini, u oddiy so‘nggi tugun (leaf certificate) ekanini bildiradi.
* `keyUsage=digitalSignature,keyEncipherment`: Kalit raqamli imzo va TLS kalit almashinuvini shifrlash uchun mo‘ljallangan.
* `extendedKeyUsage=serverAuth`: Ushbu sertifikat faqat TLS veb-serveri autentifikatsiyasi uchun yaroqlidir.
* `subjectAltName=DNS:srv1.a2.lab,IP:192.168.57.20`: **Eng muhim qator!** Mijozlar ushbu sertifikatni ham `srv1.a2.lab` domeniga, ham `192.168.57.20` IP manziliga to‘g‘ri keladi deb hisoblaydi.

#### 3.4. Server sertifikatini CA orqali imzolash va ruxsatlarni to‘g‘rilash
```bash
cd /etc/a2-tls
sudo openssl x509 -req -in srv1.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out srv1.crt -days 120 -sha256 -extfile srv1.ext
sudo chmod 600 ca.key srv1.key
sudo chmod 644 ca.crt srv1.crt
sudo install -o student -g student -m 644 ca.crt /home/student/a2-ca.crt
```
*Komandalar tahlili:*
* `openssl x509 -req`: CSR so‘rovini haqiqiy X.509 sertifikatiga aylantirish buyrug‘i.
* `-in srv1.csr`: Imzolanishi kerak bo‘lgan so‘rov fayli.
* `-CA ca.crt -CAkey ca.key`: Imzolash uchun biz yaratgan Lokal CA sertifikati va maxfiy kalitidan foydalanish.
* `-CAcreateserial`: Har bir imzolangan sertifikatga unikal seriya raqami berish uchun `ca.srl` faylini yaratadi.
* `-out srv1.crt`: Chiqariladigan tayyor server sertifikati.
* `-days 120`: Sertifikat amal qilish muddati 120 kun etib belgilanadi.
* `-extfile srv1.ext`: Tayyorlangan SAN va foydalanish cheklovlarini sertifikatga kiritadi.
* `chmod 600 ca.key srv1.key`: Maxfiy kalitlarni `rw-------` qilib, faqat `root` foydalanuvchisi o‘qiy oladigan qiladi (begonalardan himoya).
* `chmod 644 ca.crt srv1.crt`: Ommaviy sertifikatlarni barcha o‘qiy oladigan qiladi (`rw-r--r--`).
* `sudo install -o student -g student -m 644 ca.crt /home/student/a2-ca.crt`: CA sertifikatini `student` foydalanuvchisiga tegishli qilib uning uy katalogiga joylashtiradi (mijoz mashinasi osongina yuklab olishi uchun).

#### 3.5. Sertifikatni tekshirish
```bash
sudo openssl verify -CAfile ca.crt srv1.crt
sudo openssl x509 -in srv1.crt -noout -subject -issuer -dates -ext subjectAltName
```
*Komandalar tahlili:*
* `openssl verify -CAfile ca.crt srv1.crt`: Ishonch zanjirini tekshiradi. Natijada `srv1.crt: OK` chiqishi shart!
* `openssl x509 -in srv1.crt`: Sertifikat ichidagi maydonlarni inson o‘qiy oladigan formatda chiqaradi:
  - `-noout`: Sertifikatning uzun shifrlangan Base64 kodini chop etmaslik.
  - `-subject`: Sertifikat egasini ko‘rsatish (`CN=srv1.a2.lab`).
  - `-issuer`: Sertifikat kim tomonidan berilganini ko‘rsatish (`CN=A2 Lab CA`).
  - `-dates`: Amal qilish boshlanish va tugash sanasi.
  - `-ext subjectAltName`: SAN maydonida DNS va IP manzillarini tekshirish.

---

### 4-bosqich: Nginx da TLS 1.2/1.3 HTTPS xizmatini ishga tushirish va sinovlar

#### 4.1. Nginx server blokini sozlash (SRV-1 da)
Server konfiguratsiya faylini yaratamiz:
```bash
sudo nano /etc/nginx/sites-available/a2-tls
```
Fayl mazmuni:
```nginx
server {
    listen 443 ssl;
    server_name srv1.a2.lab;

    ssl_certificate     /etc/a2-tls/srv1.crt;
    ssl_certificate_key /etc/a2-tls/srv1.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        default_type text/plain;
        return 200 "A2 TLS service is working\n";
    }
}
```
*Konfiguratsiya tahlili:*
* `listen 443 ssl`: Nginx 443-portda standart SSL/TLS shifrlash bilan ulanishlarni kutadi.
* `server_name srv1.a2.lab`: Ushbu blok qaysi host nomiga javob berishini belgilaydi (SNI uchun).
* `ssl_certificate` va `ssl_certificate_key`: 3-bosqichda yaratilgan sertifikat va maxfiy kalit yo‘llari.
* `ssl_protocols TLSv1.2 TLSv1.3`: Eskirgan va zaif TLS 1.0 hamda TLS 1.1 ni taqiqlab, faqat zamonaviy xavfsiz protokollarga ruxsat beradi.
* `location /`: Bosh sahifaga so‘rov kelganda oddiy matn rejimida `200 OK` status kodi bilan `"A2 TLS service is working"` javobini qaytaradi.

Saytni faollashtiramiz va Nginx ni qayta yuklaymiz:
```bash
sudo ln -sfn /etc/nginx/sites-available/a2-tls /etc/nginx/sites-enabled/a2-tls
sudo nginx -t
sudo systemctl reload nginx
sudo ss -lntp | grep ':443'
```
*Komandalar tahlili:*
* `ln -sfn`: `sites-available` dagi fayldan `sites-enabled` ga simvolik havola (symlink) yaratadi (`-s` — symlink, `-f` — mavjud bo‘lsa ustiga yozish, `-n` — katalogni to‘g‘ri yo‘naltirish).
* `sudo nginx -t`: Nginx konfiguratsiyasining sintaksisini sinovdan o‘tkazadi (`syntax is ok, test is successful`).
* `sudo systemctl reload nginx`: Veb-serverni to‘xtatmasdan yangi sozlamani kuchga kiritadi.
* `ss -lntp | grep ':443'`: 443-portda `nginx` xizmati tinglayotganini ko‘rsatadi.

#### 4.2. CLIENT-1 da DNS va CA sertifikatini sozlash
CLIENT-1 terminaliga o‘tamiz.

1. Mahalliy DNS xaritalash (Chunki laboratoriyada tashqi DNS yo‘q):
```bash
echo '192.168.57.20 srv1.a2.lab' | sudo tee -a /etc/hosts
```
*Komanda tahlili:*
* `echo '...' | sudo tee -a /etc/hosts`: `/etc/hosts` fayliga `192.168.57.20` manzili `srv1.a2.lab` nomiga tegishli ekanligini qo‘shib qo‘yadi (`tee -a` — append rejimida root nomidan yozish).

2. CA sertifikatini serverdan xavfsiz nusxalab olish:
```bash
scp -i ~/.ssh/a2_ed25519 student@192.168.57.20:~/a2-ca.crt ~/a2-ca.crt
openssl x509 -in ~/a2-ca.crt -noout -subject -fingerprint -sha256
```
*Komandalar tahlili:*
* `scp`: SSH protokoli orqali fayllarni xavfsiz nusxalash utilitasi (Secure Copy).
* `-i ~/.ssh/a2_ed25519`: Nusxalashda biz yaratgan SSH kalitidan foydalanadi (parol so‘ramaydi).
* `openssl x509 -in ~/a2-ca.crt -noout -subject -fingerprint -sha256`: Olingan CA sertifikatining SHA-256 barmoq izini (fingerprint) hisoblaydi (fayl yo‘lda buzilmaganini tasdiqlash uchun).

#### 4.3. Ijobiy va nazoratli salbiy HTTPS sinovlari
CLIENT-1 da quyidagi 3 ta sinovni o‘tkazamiz:

1. **Ijobiy sinov (To‘g‘ri nom va ishonchli CA bilan):**
```bash
curl --cacert ~/a2-ca.crt https://srv1.a2.lab/
```
*Komanda tahlili:*
* `curl`: Veb-sahifaga so‘rov yuborish.
* `--cacert ~/a2-ca.crt`: Tizimning global sertifikatlari o‘rniga aynan bizning Lokal CA sertifikatimizga ishonishni uqtiradi.
* *Kutiladigan natija:* `A2 TLS service is working` matni chop etiladi.

2. **TLS seansini chuqur tekshirish (OpenSSL s_client):**
```bash
openssl s_client -connect 192.168.57.20:443 -servername srv1.a2.lab -CAfile ~/a2-ca.crt </dev/null 2>/dev/null | grep -E 'Protocol|Cipher|Verify return code'
```
*Komanda parametrlari:*
* `s_client`: OpenSSL ning generik SSL/TLS mijozi.
* `-connect 192.168.57.20:443`: To‘g‘ridan-to‘g‘ri server IP va portiga ulanadi.
* `-servername srv1.a2.lab`: TLS Handshake vaqtida Server Name Indication (SNI) sarlavhasini yuboradi.
* `-CAfile ~/a2-ca.crt`: Zanjirni tekshirish uchun CA sertifikatini ko‘rsatadi.
* `</dev/null 2>/dev/null`: Kirish oqimini yopadi va xatolik shovqinlarini yashiradi.
* *Kutiladigan natija:*
  ```text
  Protocol  : TLSv1.3 (yoki TLSv1.2)
  Cipher    : TLS_AES_256_GCM_SHA384
  Verify return code: 0 (ok)
  ```
  `Verify return code: 0 (ok)` chiqishi ishonch zanjiri to‘liq tasdiqlanganini bildiradi!

3. **Nazoratli salbiy sinov (Noto‘g‘ri host nomi bilan murojaat):**
```bash
curl --resolve wrong.a2.lab:443:192.168.57.20 --cacert ~/a2-ca.crt https://wrong.a2.lab/
```
*Komanda parametrlari:*
* `--resolve wrong.a2.lab:443:192.168.57.20`: DNS tizimiga murojaat qilmasdan, `wrong.a2.lab` nomini to‘g‘ridan-to‘g‘ri server IP siga yo‘naltiradi.
* *Kutiladigan natija:* `curl: (60) SSL: certificate subject name 'srv1.a2.lab' does not match target host name 'wrong.a2.lab'` xatosi chiqadi.
* *Xulosa:* Serverning IP manzili to‘g‘ri, CA sertifikati to‘g‘ri bo‘lsa ham, domen nomi sertifikatdagi **SAN** maydoniga to‘g‘ri kelmagani uchun mijoz ulanishni darhol to‘xtatadi. Bu MITM (odam-oraliqda) hujumiga qarshi mexanizmdir.

> **OGOHLANTIRISH:**
> Hech qachon testlarda `curl -k` yoki `curl --insecure` ishlatmang! Bu bayroq barcha xavfsizlik va sertifikat tekshiruvlarini o‘chirib qo‘yadi va sinovni ilmiy jihatdan ma’nosiz qiladi.

---

### 5-bosqich: strongSwan orqali IKEv2/IPsec transport rejimini sozlash

Ushbu bosqichda ikkala VM o‘rtasida tarmoq sathi (L3) shifrlashini o‘rnatamiz.

#### 5.1. Zaxira nusxalar olish (CLIENT-1 va SRV-1 da)
Har ikkala VM terminalida:
```bash
sudo cp -a /etc/ipsec.conf /etc/ipsec.conf.a2-backup
sudo cp -a /etc/ipsec.secrets /etc/ipsec.secrets.a2-backup
```

#### 5.2. /etc/ipsec.conf faylini sozlash (CLIENT-1 va SRV-1 da bir xil!)
Har ikkala VMda faylni ochamiz:
```bash
sudo nano /etc/ipsec.conf
```
Fayl mazmuni:
```text
config setup
    uniqueids=no

conn a2-host-to-host
    keyexchange=ikev2
    type=transport
    authby=psk
    left=192.168.57.10
    right=192.168.57.20
    ike=aes256-sha256-modp2048!
    esp=aes256-sha256!
    dpdaction=restart
    dpddelay=30s
    rekey=no
    auto=start
```
*Har bir parametrning mukammal tahlili:*
* `config setup`: strongSwan ning umumiy dasturiy sozlamalari bloki.
* `uniqueids=no`: Bir xil ID ga ega bir nechta ulanishga ruxsat berish (qayta ulanishlarda qulaylik).
* `conn a2-host-to-host`: Ulanish profilining nomi.
* `keyexchange=ikev2`: Zamonaviy va xavfsiz IKEv2 protokolidan foydalanish (IKEv1 eskirgan).
* `type=transport`: **Juda muhim!** Ulanish rejimi sifatida **Transport rejimi** tanlanmoqda (asl IP header saqlanadi, faqat L4 payload shifrlanadi).
* `authby=psk`: Autentifikatsiya Pre-Shared Key (oldindan kelishilgan maxfiy so‘z) orqali amalga oshiriladi.
* `left=192.168.57.10`: Birinchi tengdosh (peer) — CLIENT-1 ning IP manzili.
* `right=192.168.57.20`: Ikkinchi tengdosh (peer) — SRV-1 ning IP manzili.
  *(Eslatma: strongSwan lokal IP manzilga qarab qaysi tomon o‘zi ekanligini avtomatik aniqlaydi).*
* `ike=aes256-sha256-modp2048!`: IKE SA (1-faza) shifrlash to‘plami:
  - `aes256`: AES 256-bitli simmetrik shifrlash.
  - `sha256`: Birlik va yaxlitlik tekshiruvi uchun SHA-2 256-bit hesh.
  - `modp2048`: Diffie-Hellman 14-guruhi (2048-bit modul).
  - `!`: Faqat ko‘rsatilgan to‘plamga qat’iy ruxsat berish (boshqasini qabul qilmaslik).
* `esp=aes256-sha256!`: CHILD SA (2-faza) foydali yuklamani shifrlash to‘plami (ESP protokoli uchun AES-256 va SHA-256).
* `dpdaction=restart`: DPD (Dead Peer Detection) bo‘yicha agar qarama-qarshi tomon javob bermay qolsa, tunnelni qayta yuklash.
* `dpddelay=30s`: Tengdoshning tirikligini har 30 soniyada tekshirish.
* `rekey=no`: Laboratoriya davomida tasodifiy qayta kalitlash (rekeying) bo‘lmasligi uchun.
* `auto=start`: Xizmat ishga tushganda ulanishni avtomatik o‘rnatishga harakat qilish.

#### 5.3. /etc/ipsec.secrets faylini sozlash (CLIENT-1 va SRV-1 da bir xil!)
Har ikkala VMda faylni tahrirlaymiz:
```bash
sudo nano /etc/ipsec.secrets
```
Fayl mazmuni:
```text
192.168.57.10 192.168.57.20 : PSK "A2-Lab-Only-Change-2026!"
```
*Tahlil:* Ushbu qator `192.168.57.10` va `192.168.57.20` o‘rtasidagi IKEv2 ulanishi uchun umumiy maxfiy kalit sifatida `"A2-Lab-Only-Change-2026!"` parolini o‘rnatadi.

Fayl ruxsatini cheklaymiz va xizmatni qayta ishga tushiramiz:
```bash
sudo chmod 600 /etc/ipsec.secrets
sudo ipsec restart
sudo ipsec statusall
```
*Komandalar tahlili:*
* `sudo chmod 600 /etc/ipsec.secrets`: Maxfiy PSK kalitini faqat root foydalanuvchisi o‘qiy oladigan qiladi (aks holda strongSwan xavfsizlik sababli kalitni qabul qilmaydi).
* `sudo ipsec restart`: strongSwan demonini barcha konfiguratsiyalarni qayta o‘qigan holda restart qiladi.
* `sudo ipsec statusall`: strongSwan ning barcha faol ulanishlari, modullari va SA larini to‘liq ro‘yxatini ko‘rsatadi.

#### 5.4. CLIENT-1 dan ulanishni ko‘tarish (Initiation)
CLIENT-1 terminalida:
```bash
sudo ipsec up a2-host-to-host
ping -c 4 192.168.57.20
sudo ipsec statusall
```
*Komandalar tahlili:*
* `sudo ipsec up a2-host-to-host`: CLIENT-1 tashabbuskor (initiator) sifatida SRV-1 ga IKEv2 so‘rovini jo‘natadi va `a2-host-to-host` tunnelini ko‘taradi. Natijada `connection 'a2-host-to-host' established successfully` xabari chiqadi.
* `ping -c 4 192.168.57.20`: Aloqani tekshiradi. Bu vaqtda ICMP paketlari endi ochiq emas, ESP ichiga o‘ralgan holda boradi.
* `sudo ipsec statusall`: Holatni qayta ko‘rsatadi. Unda quyidagi yozuvlar bo‘lishi shart:
  - `Security Associations (1 up, 0 connecting)`
  - `a2-host-to-host[1]: ESTABLISHED` (IKE SA o‘rnatilgan)
  - `a2-host-to-host{1}: INSTALLED, TRANSPORT, reqid 1, ESP SPIs: c..._i c..._o` (CHILD SA o‘rnatilgan va transport rejimida).

---

### 6-bosqich: IPsec SA, Linux XFRM siyosati va tcpdump orqali shifrlangan trafik tahlili

Ushbu bosqichda IPsec shifrlashi operatsion tizim yadrosida qanday ishlayotganini tekshiramiz.

#### 6.1. Linux yadrosi XFRM quyi tizimini ko‘rish
Har ikkala VMda quyidagi buyruqlarni bajaring:
```bash
sudo ip xfrm state
sudo ip xfrm policy
```
*Chiqishlar tahlili:*
* `sudo ip xfrm state`: Linux yadrosining **SAD (Security Association Database)** jadvalini ko‘rsatadi.
  - Har bir yo‘nalish uchun alohida yozuv bo‘ladi (biri kiruvchi, biri chiquvchi).
  - Unda har bir yo‘nalishning **SPI (Security Parameter Index)** qiymati, ishlatilayotgan shifrlash algoritmi (`auth-trunc hmac(sha256)`, `enc cbc(aes)`) va yadro darajasidagi simmetrik kalitlar ko‘rinadi.
* `sudo ip xfrm policy`: Linux yadrosining **SPD (Security Policy Database)** jadvalini ko‘rsatadi.
  - Qaysi yo‘nalishdagi paketlar qanday qayta ishlanishini belgilaydi.
  - Masalan: `src 192.168.57.10 dst 192.168.57.20 ... tmpl src 192.168.57.10 dst 192.168.57.20 proto esp mode transport`. Bu yadroga "Agar paket 10 dan 20 ga ketayotgan bo‘lsa, uni ESP transport rejimida shifrlab yubor" degan qat’iy buyruqdir.

#### 6.2. tcpdump orqali ESP paketlarini tutish va tahlil qilish
Endi shifrlashni paketlar darajasida isbotlaymiz.

1. **SRV-1 terminalida** tinglovchini yoqamiz:
```bash
LAB_IF=ens37
sudo tcpdump -ni "$LAB_IF" 'esp or udp port 500 or udp port 4500' -c 10
```
*Komanda parametrlari:*
* `tcpdump`: Tarmoq paketlarini tahlil qiluvchi qudratli sniffer.
* `-n`: IP va portlarni nomlarga aylantirmasdan (DNS so‘rovlari yubormasdan) toza raqam ko‘rinishida chiqarish.
* `-i "$LAB_IF"`: Qaysi interfeysdagi paketlarni eshitishni belgilash.
* `'esp or udp port 500 or udp port 4500'`: BPF filtri! Faqat ESP protokoli (IP proto 50) hamda IKEv2 portlarini (UDP 500/4500) tutadi.
* `-c 10`: Aniq 10 ta paket tutib, dasturni avtomatik to‘xtatish.

2. **CLIENT-1 terminalidan** trafik yuboramiz:
```bash
ping -c 6 192.168.57.20
curl --cacert ~/a2-ca.crt https://srv1.a2.lab/
```

3. **Tahlil va natija:**
* SRV-1 dagi `tcpdump` oynasida quyidagi kabi qatorlar chiqadi:
  ```text
  IP 192.168.57.10 > 192.168.57.20: ESP(spi=0x1a2b3c4d,seq=1), length 104
  IP 192.168.57.20 > 192.168.57.10: ESP(spi=0x9f8e7d6c,seq=1), length 104
  ```
* **Ilmiy xulosa:**
  - Biz `ping` (ICMP) va `curl` (HTTPS) yubordik, ammo `tcpdump` da bitta ham `ICMP echo request` yoki `HTTP/TLS application data` ko‘rinmadi! Barcha ma’lumotlar **ESP** ichiga qadoqlangan.
  - Sarlavhada `192.168.57.10` va `192.168.57.20` ochiq ko‘rinib turibdi — bu aynan **Transport rejimi** ning o‘ziga xos jihatidir. Paketning foydali yuklamasi (payload) esa to‘liq shifrlangan bo‘lib, uchinchi shaxs uni o‘qiy olmaydi.

---

### 7-bosqich: Nazoratli xatolik sinovi: Noto‘g‘ri PSK va tizimni tiklash

Ushbu bosqichda ataylab xatolik kiritib, tizimning autentifikatsiya buzilishiga qanday munosabat bildirishini tekshiramiz.

#### 7.1. CLIENT-1 da noto‘g‘ri PSK kiritish
CLIENT-1 terminalida:
```bash
sudo cp -a /etc/ipsec.secrets /etc/ipsec.secrets.a2-ok
sudo ipsec down a2-host-to-host
sudo sed -i 's/A2-Lab-Only-Change-2026!/A2-Wrong-PSK-2026!/' /etc/ipsec.secrets
sudo ipsec restart
sudo ipsec up a2-host-to-host
```
*Komandalar tahlili:*
* `cp -a ... /etc/ipsec.secrets.a2-ok`: To‘g‘ri ishlayotgan faylning etalon zaxirasini oladi.
* `sudo ipsec down a2-host-to-host`: Hozirgi ishlab turgan xavfsizlik assotsiatsiyasini (SA) majburiy to‘xtatadi.
* `sudo sed -i 's/.../.../' /etc/ipsec.secrets`: `sed` oqimli tahrirlovchisi yordamida fayl ichidagi to‘g‘ri parolni `"A2-Wrong-PSK-2026!"` noto‘g‘ri paroli bilan almashtiradi.
* `sudo ipsec restart`: strongSwan ni noto‘g‘ri kalit bilan qayta yuklaydi.
* `sudo ipsec up a2-host-to-host`: Ulanishni ko‘tarishga urinadi.
* *Kutiladigan natija:* Ulanish muvaffaqiyatsiz tugaydi (`received AUTHENTICATION_FAILED notify error` yoki `peer not acceptable`).

Jurnallarni tekshirib, xatolikni qayd qilamiz:
```bash
sudo journalctl -u strongswan-starter -n 30 --no-pager
```
*Komanda parametrlari:*
* `journalctl`: Systemd jurnalini o‘qish utilitasi.
* `-u strongswan-starter`: Faqat strongSwan xizmatiga tegishli jurnallarni saralash.
* `-n 30`: Oxirgi 30 qatorni ko‘rsatish.
* `--no-pager`: Natijani to‘xtovsiz ekranga chiqarish.
* *Natija:* Jurnalda `AUTHENTICATION_FAILED` xabari aniq ko‘rinadi.

#### 7.2. Tizimni avvalgi ishchi holatiga qaytarish (Rollback)
CLIENT-1 terminalida:
```bash
sudo cp -a /etc/ipsec.secrets.a2-ok /etc/ipsec.secrets
sudo chmod 600 /etc/ipsec.secrets
sudo ipsec restart
sudo ipsec up a2-host-to-host
sudo ipsec statusall
```
*Komandalar tahlili:*
* To‘g‘ri zaxiradagi fayl o‘z joyiga qaytariladi, ruxsatlar `600` qilinadi, servis qayta ishga tushirilib, ulanish qayta o‘rnatiladi.
* `sudo ipsec statusall` orqali ulanish yana `ESTABLISHED` va `INSTALLED` bo‘lganligi tasdiqlanadi.

---

### 8-bosqich: Yakuniy integratsion tekshiruv va laboratoriyani xavfsiz yakunlash

IPsec qayta tiklangach, yuqori qatlam protokollari (SSH va HTTPS) IPsec shifrlangan kanali ustidan uzluksiz ishlayotganini isbotlaymiz:

CLIENT-1 da bajaramiz:
```bash
ssh -i ~/.ssh/a2_ed25519 student@192.168.57.20 'echo SSH_OVER_IPSEC_OK'
curl --cacert ~/a2-ca.crt https://srv1.a2.lab/
sudo ipsec statusall
sudo ip xfrm policy
```
*Tahlil:*
* SSH ulanishi `SSH_OVER_IPSEC_OK` javobini beradi.
* HTTPS ulanishi `A2 TLS service is working` javobini beradi.
* `ipsec statusall` tunnel faol ekanini tasdiqlaydi.
* **Xulosa:** Ushbu sinov tarmoq sathi (L3) da IPsec to‘liq shifrlashni amalga oshirayotgan paytda ham yuqori sathdagi (L4-L7) TLS va SSH protokollari hech qanday to‘siqsiz, shaffof (transparent) va ko‘p qatlamli himoyalangan tarzda ishlashini amalda isbotlaydi.

**Laboratoriyani yakunlash:**
Mashg‘ulot tugagach, VMlarni to‘xtatib (`sudo poweroff`), o‘qituvchi ko‘rsatmasiga binoan boshlang‘ich snapshot holatiga qaytiladi. Maxfiy kalitlar va parollar ochiq qoldirilmaydi.

---

### 9-bosqich: Natija jadvallari va o‘qituvchi himoya savollariga to‘liq javoblar

#### 9.1. Laboratoriya va xizmatlar holati jadvali (3.1-jadval)

| Obyekt | Interfeys | IP manzil / Niqob | Xizmatlar / Portlar | Bazaviy holat |
| :--- | :--- | :--- | :--- | :--- |
| **CLIENT-1** | `ens37` (yoki `LAB_IF`) | `192.168.57.10/24` | OpenSSH client, curl, strongSwan initiator | UP, Gateway/DNS yo‘q |
| **SRV-1** | `ens37` (yoki `LAB_IF`) | `192.168.57.20/24` | 22/tcp (SSH), 443/tcp (HTTPS), UDP 500/4500 (IKE), IP 50 (ESP) | UP, Gateway/DNS yo‘q |
| **A2-LAB** | VMware `vmnet2` | `192.168.57.0/24` | Izolyatsiyalangan L2/L3 virtual tarmoq segmenti | DHCP yo‘q, Host-only |

#### 9.2. SSH sozlamasi va sinovlari jadvali (3.2-jadval)

| Tekshiruv bosqichi | Kutiladigan natija | Haqiqiy natija | Holati |
| :--- | :--- | :--- | :--- |
| `sshd -t` | Sintaksis xatoliksiz tugaydi | Chiqish bo‘sh (kod: 0) | Muvaffaqiyatli |
| `PubkeyAuthentication` | `yes` | `pubkeyauthentication yes` | Muvaffaqiyatli |
| `PasswordAuthentication`| `no` | `passwordauthentication no` | Muvaffaqiyatli |
| `PermitRootLogin` | `no` | `permitrootlogin no` | Muvaffaqiyatli |
| Kalitli kirish sinovi | `SSH_KEY_OK` qaytadi | `SSH_KEY_OK` | Muvaffaqiyatli |
| Password-only urinish | `Permission denied (publickey)` | `Permission denied (publickey)` | Muvaffaqiyatli (Nazoratli rad) |

#### 9.3. TLS sertifikati va HTTPS sinovlari jadvali (3.3-jadval)

| Maydon / Sinov turi | Kutiladigan qiymat | Haqiqiy qiymat | Holati |
| :--- | :--- | :--- | :--- |
| **Subject** | `CN=srv1.a2.lab` | `subject=C=UZ, O=A2 Lab, CN=srv1.a2.lab` | To‘g‘ri |
| **Issuer** | `CN=A2 Lab CA` | `issuer=C=UZ, O=A2 Lab, CN=A2 Lab CA` | To‘g‘ri |
| **SAN (Subject Alt Name)**| `DNS:srv1.a2.lab, IP:192.168.57.20` | `DNS:srv1.a2.lab, IP:192.168.57.20` | To‘g‘ri |
| **Amal qilish muddati** | 120 kun | `notBefore=... notAfter=... (120 days)` | To‘g‘ri |
| `openssl verify` | `OK` | `srv1.crt: OK` | Muvaffaqiyatli |
| To‘g‘ri host (`srv1.a2.lab`) | `200 OK`, `Verify return code: 0` | `A2 TLS service is working`, code 0 | Muvaffaqiyatli |
| Noto‘g‘ri host (`wrong.a2.lab`) | Sertifikat nomi mos emas (Xato 60) | `SSL: certificate subject name does not match` | Muvaffaqiyatli (Nazoratli rad) |

#### 9.4. IPsec SA va XFRM tekshiruvi jadvali (3.4-jadval)

| Tekshiruv turi | Kutiladigan natija | Haqiqiy kuzatuv | Holati |
| :--- | :--- | :--- | :--- |
| **IKE SA** | `ESTABLISHED` | `1 up, 0 connecting, a2-host-to-host[1]: ESTABLISHED` | Faol |
| **CHILD SA** | `INSTALLED, TRANSPORT` | `a2-host-to-host{1}: INSTALLED, TUNNEL=no (TRANSPORT)`| Faol |
| **Peer / Selektorlar** | `192.168.57.10 === 192.168.57.20` | `192.168.57.10/32 === 192.168.57.20/32` | To‘g‘ri |
| **IKE algoritmlari** | `AES-256 / SHA-256 / MODP-2048` | `AES_CBC_256/HMAC_SHA2_256_128/MODP_2048` | Mos |
| **ESP algoritmlari** | `AES-256 / SHA-256` | `AES_CBC_256/HMAC_SHA2_256_128` | Mos |
| `ip xfrm state` | Kiruvchi va chiquvchi ESP SAD | 2 ta ESP state yozuvi mavjud (in/out SPI) | To‘g‘ri |
| `ip xfrm policy` | in/out transport siyosatlari | Siyosatlar yadroga yuklangan (`mode transport`) | To‘g‘ri |
| `tcpdump` tahlili | ICMP ochiq ko‘rinmaydi, faqat ESP | `IP ... > ...: ESP(spi=0x...,seq=...)` | Tasdiqlandi |

#### 9.5. Integratsion va nazoratli salbiy sinovlar jadvali (3.5-jadval)

| Test ID | Ssenariy tavsifi | Kutiladigan natija | Haqiqiy natija | Xulosa va Izoh |
| :---: | :--- | :--- | :--- | :--- |
| **T1** | SSH kalitli kirish | Muvaffaqiyatli kirish | `SSH_KEY_OK` | ED25519 kaliti to‘g‘ri tekshirildi |
| **T2** | SSH parolli urinish | Ulanish rad etilishi | `Permission denied` | Parol orqali kirish bloklangan |
| **T3** | TLS to‘g‘ri SAN | 200 OK va Verify OK | Sahifa ochildi | Lokal CA va SAN to‘liq tasdiqlandi |
| **T4** | TLS noto‘g‘ri host | Sertifikat xatosi | `curl: (60) mismatch`| MITM hujumidan himoya ishlamoqda |
| **T5** | IPsec to‘g‘ri PSK | SA o‘rnatiladi | `INSTALLED` | IKEv2 muvaffaqiyatli kelishildi |
| **T6** | IPsec noto‘g‘ri PSK | Autentifikatsiya xatosi | `AUTHENTICATION_FAILED`| Begona ulanish rad etildi |
| **T7** | Tiklashdan so‘ng SSH+HTTPS | Ikkisi ham ishlaydi | Ikkisi ham muvaffaqiyatli| IPsec ustida L7 protokollari shaffof ishlaydi |

---

### O‘qituvchi uchun himoya savollariga batafsil ilmiy javoblar

1. **SSH, TLS va IPsec OSI/TCP-IP modelining qaysi qamrovlarida himoya beradi?**
   * *Javob:* IPsec tarmoq sathida (L3 - Network layer) ishlaydi va operatsion tizim yadrosida barcha yuqori qatlam IP paketlarini ilovadan qat’i nazar shifrlaydi. TLS transport va amaliy qatlam oralig‘ida (L4–L7) ishlaydi va faqat o‘ziga moslashtirilgan dastur sessiyasini (masalan, HTTPS) himoyalaydi. SSH esa amaliy qatlamda (L7 - Application layer) ishlaydi va o‘zining terminal hamda ma’lumot uzatish (SFTP) oqimlarini himoyalaydi.
2. **SSH host kaliti bilan foydalanuvchi kalitining vazifasi nimasi bilan farq qiladi?**
   * *Javob:* Host kaliti serverning o‘ziga tegishli bo‘lib, mijozga serverning haqiqiyligini isbotlaydi (server soxtalashtirilmaganini tekshirish uchun). Foydalanuvchi kaliti esa mijozga tegishli bo‘lib, serverga ma’lum bir foydalanuvchining shaxsini tasdiqlash (autentifikatsiya) uchun xizmat qiladi.
3. **PasswordAuthentication no dan oldin nima uchun ikkinchi SSH sessiyasini sinash kerak?**
   * *Javob:* Agar konfiguratsiyada xatolik bo‘lsa yoki kalit serverga to‘liq o‘rnatilmagan bo‘lsa, parolni o‘chirib SSH ni reload qilgach, administrator serverdan butunlay uzilib qolishi (lockout) mumkin. Mavjud sessiyani yopmasdan ikkinchi sessiyani sinash — xatolik yuz berganda serverni zaxiradan zudlik bilan tiklash imkoniyatini beradi.
4. **TLS sertifikatidagi Subject, Issuer va SAN maydonlari qanday vazifa bajaradi?**
   * *Javob:* `Subject` — sertifikat kimga tegishli ekanini (server domen nomi); `Issuer` — ushbu sertifikatni qaysi sertifikatlash markazi (CA) imzolaganini ko‘rsatadi; `SAN (Subject Alternative Name)` — sertifikat haqiqiy hisoblangan barcha muqobil domen nomlari va IP manzillar ro‘yxatini belgilaydi.
5. **curl --cacert muvaffaqiyati nimani isbotlaydi, --insecure esa nega yaroqsiz dalil?**
   * *Javob:* `curl --cacert` mijoz faqatgina berilgan CA zanjiriga qat’iy tayangan holda server sertifikatining butunligini, muddati va SAN mosligini tekshirganini isbotlaydi. `--insecure` (`-k`) esa barcha kriptografik tekshiruvlarni butunlay o‘chirib qo‘yadi, natijada soxta yoki o‘g‘irlangan sertifikatlar ham qabul qilinib, xavfsizlik nolga tenglashadi.
6. **wrong.a2.lab sinovi nima uchun rad etildi, garchi IP manzil va CA to‘g‘ri bo‘lsa ham?**
   * *Javob:* Chunki TLS spetsifikatsiyasiga ko‘ra mijoz murojaat qilayotgan host nomi server taqdim etgan sertifikatning SAN maydonidagi nomlarga aniq mos kelishi shart. `wrong.a2.lab` sertifikatda mavjud bo‘lmagani sababli mijoz buni MITM (DNS soxtalashtirish) hujumi deb baholaydi va aloqani uzadi.
7. **IKE SA bilan CHILD SA o‘rtasidagi farq nima?**
   * *Javob:* IKE SA (1-faza) ikkita tengdosh o‘rtasida xavfsiz boshqaruv kanalini yaratadi, ularni autentifikatsiya qiladi va umumiy kalitlarni kelishadi. CHILD SA (2-faza) esa bevosita haqiqiy foydali yuklamani (tarmoq trafigini) ESP protokoli yordamida shifrlab uzatish uchun yadro darajasidagi yo‘naltirilgan xavfsizlik parametrlarini o‘rnatadi.
8. **Transport rejimida qaysi ma’lumot ochiq qoladi va qaysi qism ESP bilan himoyalanadi?**
   * *Javob:* Transport rejimida asl IP sarlavhasi (manba va qabul qiluvchi IP manzillari) ochiq qoladi. Asl IP sarlavhasidan keyingi barcha ma’lumotlar — TCP/UDP/ICMP sarlavhalari va ilovaning foydali yuklamasi to‘liq ESP yordamida shifrlanadi.
9. **ip xfrm state va ip xfrm policy chiqishlari bir-biridan nimani ko‘rsatadi?**
   * *Javob:* `ip xfrm state` (SAD) — shifrlashning operativ holatini, har bir yo‘nalishdagi SPI raqamlarini, amaldagi kalitlarni va shifr algoritmlarini ko‘rsatadi. `ip xfrm policy` (SPD) esa — qaysi manba va manzillar orasidagi paketlarga nisbatan qanday shifrlash siyosati qo‘llanilishi shartligini (qoidalarni) ko‘rsatadi.
10. **tcpdump da ESP ko‘rinishi foydali yuk shifrlanganini qanday asoslaydi va nimani isbotlamaydi?**
    * *Javob:* ESP paketining ko‘rinishi L4 protokoli (ICMP/TCP) va ma’lumotlar shifrlanib, faqat ESP sarlavhasi va shifrlangan payload mavjudligini isbotlaydi. Ammo u simmetrik kalitlarning qanchalik mustahkamligini yoki tizim ichidagi dasturiy zaifliklar yo‘qligini isbotlay olmaydi.
11. **Noto‘g‘ri PSK da ulanish nega o‘rnatilmadi va tiklash muvaffaqiyatini qanday tekshirdingiz?**
    * *Javob:* Chunki IKEv2 autentifikatsiya fazasida tomonlar hisoblagan autentifikatsiya heshlari (AUTH payload) o‘zaro mos kelmadi va tizim `AUTHENTICATION_FAILED` xatosi bilan aloqani rad etdi. Tiklashdan so‘ng `sudo ipsec statusall` orqali ulanish yana `ESTABLISHED` va `INSTALLED` holatiga kelgani va ping o‘tganligi orqali muvaffaqiyat tasdiqlandi.
12. **Real muhitda IKEv2 PSK o‘rniga sertifikatli autentifikatsiyani qachon tanlaysiz?**
    * *Javob:* Real ishlab chiqarish muhitida tugunlar soni ko‘p bo‘lganda (barcha joyga bir xil PSK tarqatish xavfli), xodimlar yoki serverlar dinamik qo‘shilib-ayrilib turganda, sertifikatlarni markazlashtirilgan tarzda bekor qilish (Revocation / CRL / OCSP) zarur bo‘lganda va yuqori xavfsizlik talab etilgan korporativ tarmoqlarda sertifikatli autentifikatsiya (PKI) tanlanadi.

---

## 4-BO‘LIM: Foydalanilgan Rasmiy Manbalar va Standartlar

Ushbu qo‘llanma quyidagi xalqaro standartlar va rasmiy texnik hujjatlar asosida tuzilgan:

1. **Internet Engineering Task Force (IETF) RFC Standartlari:**
   * **RFC 8446:** The Transport Layer Security (TLS) Protocol Version 1.3 — [https://datatracker.ietf.org/doc/html/rfc8446](https://datatracker.ietf.org/doc/html/rfc8446)
   * **RFC 5280:** Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation List (CRL) Profile — [https://datatracker.ietf.org/doc/html/rfc5280](https://datatracker.ietf.org/doc/html/rfc5280)
   * **RFC 7296:** Internet Key Exchange Protocol Version 2 (IKEv2) — [https://datatracker.ietf.org/doc/html/rfc7296](https://datatracker.ietf.org/doc/html/rfc7296)
   * **RFC 4303:** IP Encapsulating Security Payload (ESP) — [https://datatracker.ietf.org/doc/html/rfc4303](https://datatracker.ietf.org/doc/html/rfc4303)
   * **RFC 4253:** The Secure Shell (SSH) Transport Layer Protocol — [https://datatracker.ietf.org/doc/html/rfc4253](https://datatracker.ietf.org/doc/html/rfc4253)
   * **RFC 8709:** Ed25519 and Ed448 Public Key Algorithms for the Secure Shell (SSH) Protocol — [https://datatracker.ietf.org/doc/html/rfc8709](https://datatracker.ietf.org/doc/html/rfc8709)

2. **Dasturiy Ta’minotlarning Rasmiy Texnik Qo‘llanmalari:**
   * **Nginx Documentation:** Configuring HTTPS servers — [https://nginx.org/en/docs/http/configuring_https_servers.html](https://nginx.org/en/docs/http/configuring_https_servers.html)
   * **OpenSSH Official Manual:** `sshd_config(5)` and Secure Shell Architecture — [https://man.openbsd.org/sshd_config](https://man.openbsd.org/sshd_config)
   * **OpenSSL Cryptography Toolkit:** `openssl-req(1)` and `openssl-x509(1)` Manuals — [https://docs.openssl.org/](https://docs.openssl.org/)
   * **strongSwan IPsec Architecture:** Configuration Quickstart and IPsec Protocol Documentation — [https://docs.strongswan.org/docs/latest/](https://docs.strongswan.org/docs/latest/)
   * **Linux Kernel IPsec (XFRM):** Linux Advanced Routing & Traffic Control and XFRM Subsystem — [https://wiki.linuxfoundation.org/networking/xfrm](https://wiki.linuxfoundation.org/networking/xfrm)
   * **Ubuntu Server Official Documentation:** Network Configuration with Netplan and OpenSSH Security — [https://ubuntu.com/server/docs](https://ubuntu.com/server/docs)
