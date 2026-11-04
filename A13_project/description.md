# TARMOQ XAVFSIZLIGI | A13: PROXY VA WAF VOSITALARINI SOZLASH
## 1-Variant: Apache Reverse Proxy, ModSecurity 2.x va OWASP Core Rule Set (CRS) Bo‘yicha To‘liq Amaliy Qo‘llanma

---

## 📌 ISHCHI MUHIT VA FOYDALANUVCHI SHARTLARI (LOKAL WI-FI VA UBUNTU HOST + VMWARE)

Ushbu qo‘llanma laboratoriya topshirig‘ini bajaruvchi talabaning real ishchi muhitiga qat’iy moslashtirilgan:

1. **Tarmoq Muhiti — Shaxsiy Lokal / Uy Wi-Fi (Universitet Tarmog‘i EMAS):**
   - Siz universitet Wi-Fi tarmog‘idan emas, **o‘zingizning shaxsiy lokal Wi-Fi (yoki uy) tarmog‘ingizdan** foydalanasiz.
   - Barcha laboratoriya sinovlari tashqi lokal Wi-Fi tarmog‘idan to‘liq izolyatsiya qilingan **VMware Host-Only (`vmnet2` yoki `vmnet1`)** segmentida (`10.130.10.0/24`) amalga oshiriladi.
   - **Xavfsizlik kafolati:** Virtual laboratoriya ichidagi WAF sinovlari, test hujumlari (XSS matnlari) va tarmoq paketlari sizning real uy Wi-Fi routeringizga yoki xonadondagi boshqa qurilmalarga umuman chiqmaydi. Uy tarmog‘ingizdagi IP manzillar (masalan, `192.168.1.0/24` yoki `192.168.0.0/24`) bilan to‘qnashuv (IP conflict) yuzaga kelmaydi.

2. **Asosiy Tizim (Host OS) — Ubuntu Linux Desktop (Windows Host EMAS!):**
   - Asosiy kompyuter operatsion tizimi — **Ubuntu Linux** (Desktop).
   - Virtualizatsiya muhiti: Ubuntu ichiga o‘rnatilgan **VMware Workstation Pro / Player for Linux**.
   - Laboratoriyada mijoz (Client) sifatida alohida Windows mashinasi talab etilmaydi! Barcha mijoz so‘rovlari, test skriptlari va tekshiruvlar to‘g‘ridan-to‘g‘ri **Ubuntu Host terminali** (Bash, Linux `curl`, `nc` / `netcat`) orqali amalga oshiriladi (laboratoriya qo‘llanmasidagi Windows PowerShell `curl.exe` va `Test-NetConnection` buyruqlarining barchasi Linux ekvivalentlariga to‘liq o‘girib berilgan).

3. **Tarmoq Topologiyasi va IP Arxitekturasi:**
   - **Mijoz (Ubuntu Host):** VMware Host-Only virtual adapteri (`vmnet2` / `vmnet1`) — IP manzili: `10.130.10.1/24`.
   - **WAF va Reverse Proxy Serveri (WAF-SRV VM):** Ubuntu Server 24.04 LTS — IP manzili: `10.130.10.10/24` (Tashqi tarmoq uchun faqat Apache 2.4 serverining TCP 80 porti ochiq).
   - **Backend Servis (Ichki Ilova):** WAF-SRV serverining o‘zida joylashgan Python HTTP ilovasi — IP manzili: `127.0.0.1:8080` (**Loopback**; tashqi tarmoq interfeyslarida tinglamaydi, tashqaridan to‘g‘ridan-to‘g‘ri ulanish butunlay imkonsiz).

```
+-----------------------------------------------------------------------------------------+
| UBUNTU HOST (Asosiy kompyuter - Mijoz)                                                  |
| IP: 10.130.10.1/24 (vmnet2/vmnet1 interfeysi)                                           |
| Asboblar: Linux curl, nc (netcat), Bash terminal                                        |
+--------------------------------------------+--------------------------------------------+
                                             | VMware Host-Only Network (10.130.10.0/24)
                                             | (DHCP o'chiq, uy Wi-Fi tarmog'idan izolyatsiyalangan)
                                             v
+-----------------------------------------------------------------------------------------+
| WAF-SRV VM (Ubuntu Server 24.04 LTS)                                                    |
| Lab Interfeysi: 10.130.10.10/24                                                         |
|                                                                                         |
|  [ Apache 2.4 Reverse Proxy + ModSecurity 2.x WAF + OWASP CRS ] : Tinglaydi TCP 80 da    |
|      |                                                                                  |
|      | Lokal proksilash (Faqat server ichki xotirasida - Loopback)                      |
|      v                                                                                  |
|  [ Python O'quv Backend Ilovasi ] : Tinglaydi 127.0.0.1:8080 (Loopback)                 |
|  (Tashqi dunyodan to'liq yashirilgan, to'g'ridan-to'g'ri ulanish bloklangan)           |
+-----------------------------------------------------------------------------------------+
```

---

## 1. MAVZUNING ASOSIY MAQSADI VA NAZARIY ASOSLARI

### 1.1. Laboratoriya mashg‘ulotining maqsadi
Mazkur amaliy mashg‘ulotning asosiy maqsadi — veb-xizmatlarni tarmoq va dasturiy darajadagi tahdidlardan himoya qilish uchun zamonaviy arxitekturaviy yondashuvlarni o‘rganishdan iborat:
1. **Teskari proksi (Reverse Proxy)** vositasini (Apache 2.4) veb-xizmat oldiga joylashtirib, ichki resurslarni (origin backend) tashqi tarmoqdan to‘liq yashirish.
2. Dasturiy qatlam xavfsizlik devori — **WAF (Web Application Firewall - ModSecurity 2.x)** dvigatelini o‘rnatish va unga xalqaro standart darajasidagi **OWASP Core Rule Set (CRS)** xavfsizlik qoidalari to‘plamini integratsiya qilish.
3. WAF tizimining ikki xil ishlash rejimini amalda solishtirish:
   - **Kuzatuv rejimi (`DetectionOnly`):** Hujum belgilarini aniqlab logga yozadi, ammo so‘rovni bloklamaydi (HTTP 200).
   - **Faol bloklash rejimi (`On`):** Shubhali so‘rovlarni darhol rad etadi (HTTP 403 Forbidden).
4. **Soxta ijobiy (False Positive)** holatini tahlil qilish: Qonuniy biznes trafigi (masalan, foydalanuvchining “JavaScript darsi” degan izohi) xavfsizlik qoidasi tomonidan xato ravishda bloklanganda, butun WAFni o‘chirib yubormasdan, faqatgina zarur endpoint, HTTP usuli va parametr doirasida **tor runtime istisno (fine-grained rule exclusion)** yaratish va qolgan himoyani to‘liq saqlab qolish.
5. So‘rov yo‘li, javob sarlavhalari (`ProxyPassReverse`), tranzaksiya identifikatorlari (`UNIQUE_ID`) va jurnal (audit/error/access/journal) yozuvlarini o‘zaro korrelyatsiya qilish orqali xavfsizlik arxitekturasini isbotlash.

---

### 1.2. Asosiy tushunchalar va xavfsizlik mexanizmlari

#### A) Forward Proxy va Reverse Proxy farqi
- **Forward Proxy (To‘g‘ridan-to‘g‘ri proksi):** Mijozlar (masalan, ofis xodimlari) tomonida turadi. Mijoz qaysi tashqi saytga kirishni xohlasa, so‘rovni forward proxy orqali yuboradi. Maqsadi: xodimlarning internetga chiqishini nazorat qilish, keshlashtirish yoki tashqi manzilni yashirish. Apache’da bu rejim `ProxyRequests On` orqali yoqiladi. Agar u noto‘g‘ri ochiq qoldirilsa, xakerlar serveringizni bepul "Open Proxy" sifatida suiiste’mol qilishi mumkin.
- **Reverse Proxy (Teskari proksi):** Serverlar (backend) oldida turadi. Mijoz faqat proksi manzilini biladi va unga murojaat qiladi; reverse proxy esa so‘rovni o‘z ortidagi ichki backend serverlarga taqsimlaydi va javobni mijozga qaytaradi. Apache’da bu `ProxyRequests Off` (ochiq proksi o‘chiq) va `ProxyPass` / `ProxyPassReverse` direktivalari orqali ishlaydi. Ushbu laboratoriyada aynan **Reverse Proxy** qo‘llaniladi.

#### B) Web Application Firewall (WAF) va L3/L4 Firewall farqi
- **An’anaviy Firewall (masalan, iptables, nftables, UFW):** OSI modelining 3- (Tarmoq) va 4- (Transport) qatlamlarida ishlaydi. U faqat IP-manzillar va TCP/UDP portlarni tekshiradi. Agar 80 yoki 443 port ochiq bo‘lsa, u port orqali kelayotgan HTTP so‘rovining ichida qanday zararli buyruq borligini tushunmaydi.
- **WAF (Web Application Firewall):** OSI modelining 7- (Ilova / Application) qatlamida ishlaydi. U HTTP/HTTPS trafigining ichini — URL, URI parametrlari, sarlavhalar (Headers), Cookies va POST so‘rovining tanasini (Request Body) chuqur tahlil qiladi (Deep Packet Inspection). SQL Injection (SQLi), Cross-Site Scripting (XSS), Remote Code Execution (RCE), Path Traversal kabi hujumlarni aniqlaydi va to‘xtatadi.

#### C) ModSecurity dvigatelining ishlash fazalari (Phases)
ModSecurity har bir HTTP so‘rovi va javobini 5 ta alohida bosqichda tahlil qiladi:
- **Phase 1 (Request Headers):** So‘rov sarlavhalari (Request Line, Headers, Cookies) o‘qilgach darhol bajariladi. Bu yerda mijoz IPsi, User-Agent yoki xavfsizlik markerlari tekshiriladi.
- **Phase 2 (Request Body):** So‘rov tanasi (POST parametrlari, JSON, forma ma’lumotlari) qabul qilib bo‘lingach ishga tushadi. Asosiy kiritma tekshiruvlari shu yerda kechadi. Buning uchun `SecRequestBodyAccess On` bo‘lishi shart!
- **Phase 3 (Response Headers):** Backend qaytargan javob sarlavhalari tekshiriladi.
- **Phase 4 (Response Body):** Backend qaytargan sahifa tanasi tekshiriladi (masalan, tizimdan maxfiy xatolar yoki ma’lumotlar sizib chiqmasligini nazorat qilish).
- **Phase 5 (Logging):** Tranzaksiya yakunlangach, barcha ma’lumotlar audit va xatolar jurnaliga yoziladi.

#### D) OWASP Core Rule Set (CRS) va Anomaliya Ballari (Anomaly Scoring)
- OWASP CRS — bu ModSecurity dvigateli uchun xalqaro mutaxassislar tomonidan ishlab chiqilgan qoidalar to‘plamidir.
- **Anomaliya bali qanday ishlaydi?** An’anaviy sodda qoidalar hujum belgisini ko‘rsa darhol bloklaydi. CRS esa ancha aqlliroq: u har bir aniqlangan shubhali belgi uchun so‘rovga anomaliya bali (Score) beradi (masalan, Critical=5, Error=4, Warning=3, Notice=2).
- Masalan, `941xxx` seriyasidagi qoidalar XSS belgilarini topadi va `TX:anomaly_score` o‘zgaruvchisiga ball qo‘shadi.
- Phase 2 oxirida maxsus baholovchi qoida — **Rule `949110`** ishga tushadi. U umumiy to‘plangan ball belgilangan chegaradan (odatda 5 balldan) oshganini tekshiradi va agar oshgan bo‘lsa, yakuniy bloklash amalini (`deny`, HTTP 403) qo‘llaydi.
- Demak: `941xxx` — bu **aniqlash qoidasi (detection rule)**, `949110` esa **yakuniy bloklash qoidasi (blocking evaluation rule)** hisoblanadi!

#### E) False Positive (Soxta Ijobiy) va Qoidalarni Nozik Sozlash (Tuning)
- **False Positive:** Xavfsizlik tizimi zararsiz, qonuniy foydalanuvchi ma’lumotini xato tarzda "hujum" deb qabul qilib, bloklab qo‘yishi.
- **Katta xato:** Ko‘p tajribasiz ma’murlar bunday holatda sayt ishlamay qolmasligi uchun butun WAFni o‘chirib qo‘yishadi (`SecRuleEngine Off`). Bu tizimni himoyasiz qoldiradi!
- **To‘g‘ri yechim (Tuning):** Butun qoidani ham, WAFni ham o‘chirmasdan, faqat muammo yuzaga kelgan aniq endpoint (`/submit`), aniq HTTP metodi (`POST`), aniq qoida ID (`1001301`) va aniq parametr (`ARGS:comment`) uchun tor istisno yaratish (`ctl:ruleRemoveTargetById`).

---

## 2. HAR BIR MAVZU VA KOMPONENT BO‘YICHA TO‘LIQ QO‘LLANMA

### 2.1. Backend Servis: Loopback Izolyatsiyasi
- Backend ilova — Python `http.server` asosidagi kichik REST-servis bo‘lib, u `/opt/a13/backend.py` manzilida joylashadi.
- U faqat `127.0.0.1:8080` manzilida tinglaydi. Loopback manzili faqat operatsion tizimning ichki xotirasida mavjud bo‘lib, tashqi tarmoq kartasidan (NIC) bu manzilga to‘g‘ridan-to‘g‘ri ulanib bo‘lmaydi.
- Xavfsizlik chorasi sifatida backend ilovani hech qachon `0.0.0.0:8080` manziliga bog‘lamaslik kerak. Aks holda xakerlar Apache va ModSecurity WAFni chetlab o‘tib, backendga to‘g‘ridan-to‘g‘ri so‘rov yuborishi mumkin bo‘ladi (**WAF Bypass**).
- Backend so‘rovlarni qabul qilib, ularni hech qanday operatsion tizim buyrug‘i yoki SQL buyrug‘i sifatida bajarmaydi, balki kelgan ma’lumotlarni sof JSON formatida qaytaradi.

### 2.2. Apache Reverse Proxy Mexanizmi
Apache virtual hostida quyidagi muhim xavfsizlik direktivalari joriy qilinadi:
1. `ProxyRequests Off`: Apache-ni ochiq (forward) proksi sifatida ishlashini qat’iyan taqiqlaydi.
2. `ProxyPreserveHost On`: Mijoz so‘ragan asl domen nomini (`Host` sarlavhasini) backendga o‘zgartirmasdan uzatadi.
3. `ProxyPass / http://127.0.0.1:8080/ connectiontimeout=2 timeout=5`: Barcha kiruvchi HTTP so‘rovlarni loopbackdagi Python backendga yo‘naltiradi. Agar backend 2 soniya ichida javob bermasa yoki ulanish uzilsa, uzoq kutib turmay xatolik qaytaradi.
4. `ProxyPassReverse / http://127.0.0.1:8080/`: Backend qaytargan HTTP javob sarlavhalarini (masalan, `Location`, `Content-Location`, `URI`) mijozga tushunarli bo‘lgan tashqi proksi manziliga (`http://10.130.10.10/`) moslab qayta yozadi. Agar backend 302 Redirect berib, ichki manzilni ko‘rsatsa (`Location: http://127.0.0.1:8080/landing`), bu direktiva uni `Location: http://10.130.10.10/landing` ga aylantiradi va ichki tarmoq strukturasini fosh bo‘lishdan asraydi!
5. `RequestHeader unset X-Forwarded-For` va `RequestHeader unset X-Real-IP`: Mijoz tomondan soxtalashtirib yuborilishi mumkin bo‘lgan sarlavhalarni tozalab tashlaydi.
6. `Header always set X-A13-Gateway "apache-waf"`: Barcha javoblarga maxsus sarlavha qo‘shadi — bu mijozga javob to‘g‘ridan-to‘g‘ri backenddan emas, aynan himoyalangan Apache shlyuzidan qaytayotganini isbotlaydi.

### 2.3. ModSecurity v2 Konfiguratsiyasi va Direktivalari
ModSecurity dvigatelining asosiy sozlamalari:
- `SecRuleEngine DetectionOnly | On`:
  - `DetectionOnly`: WAF qoidalari ishlaydi, tahdidlarni tekshiradi, anomaliya ballarini hisoblaydi, jurnallarga yozadi, ammo qoidadagi `deny` amalini bajarmaydi (so‘rov o‘tkazib yuboriladi, mijoz 200 oladi). Bu rejim yangi tizimlarni joriy qilishda tizimga zarar yetkazmasdan qoidalarni tekshirish uchun zarur.
  - `On`: To‘liq faol rejim. Qoida buzilgan taqdirda so‘rov darhol to‘xtatiladi va mijozga 403 Forbidden qaytariladi.
- `SecRequestBodyAccess On`: ModSecurity mijoz yuborgan POST so‘rovining tanasini xotiraga o‘qishi va tekshirishi uchun shart. Agar bu parametr `Off` bo‘lsa, veb-formalardan yuborilgan hech qanday ma’lumot (shu jumladan SQLi yoki XSS) tekshirilmay o‘tib ketadi!
- `SecAuditEngine RelevantOnly`: Faqat qoidaga tushgan yoki server xatoligi yuz bergan (4xx, 5xx) so‘rovlar audit jurnaliga batafsil yoziladi.
- `SecAuditLogParts ABFHZ`: Audit logida qaysi qismlar saqlanishini belgilaydi:
  - `A`: Tranzaksiya sarlavhasi (vaqt, Unique ID, manba IP, port).
  - `B`: Mijozning to‘liq HTTP so‘rov sarlavhalari.
  - `F`: Server qaytargan HTTP javob sarlavhalari.
  - `H`: WAF qoidasi ishlash tafsilotlari (qaysi qoida tetiklandi, qaysi qator yoki parametr sabab bo‘ldi).
  - `Z`: Tranzaksiya yakuni belgisi.

### 2.4. Qoidalarni Yuklash Tartibi (Include Hierarchy)
Apache’da ModSecurity qoidalarining yuklanish tartibi o‘ta muhim:
1. `modsecurity.conf` — Asosiy dvigatel parametrlari.
2. `a13-before.conf` — **Runtime istisnolar (Exclusions).** U CRS va mahalliy qoidalardan OLDIN yuklanishi shart! Shundagina u tekshirilayotgan qoidadan ma’lum parametrlarni olib tashlay oladi.
3. `crs-setup.conf` — OWASP CRS global sozlamalari (chegaraviy ballar, rejimlar).
4. `rules/*.conf` — CRS xavfsizlik qoidalari.
5. `a13-local.conf` — Mahalliy (maxsus yozilgan) qoidalar.

---

## 3. MASHQNI BAJARISH BO‘YICHA KETMA-KET QO‘LLANMA VA HAR BIR CLI BUYRUQNING BATAFSIL TAHLILI

### BOSQICH 1: VMware Tarmoq Muhiti va Tizim Paketlarini O‘rnatish

#### 1.1. VMware Workstation (Ubuntu Host) sozlamalari:
Ubuntu host tizimingizda VMware Workstation dasturini oching:
1. **Virtual Network Editor** menyusiga kiring (`vmware-netcfg` yoki GUI orqali).
2. Yangi **Host-Only** virtual tarmog‘ini tanlang yoki yarating (masalan, `vmnet2`).
3. Subnet IP qismiga: `10.130.10.0`, Subnet mask qismiga: `255.255.255.0` (`/24`) yozing.
4. **Use local DHCP service to distribute IP address to VMs** belgisini **O‘CHIRING** (DHCP o‘chiq bo‘lishi kerak, barcha manzillar statik beriladi).
5. **Connect a host virtual adapter to this network** belgisini yoqing (Ubuntu host bu tarmoqda `10.130.10.1` manzilini oladi).
6. Ubuntu Server virtual mashinasining tarmoq adapterini (Network Adapter) aynan shu `vmnet2` (Host-Only) tarmog‘iga ulang.

#### 1.2. Ubuntu Hostda tarmoq interfeysini tekshirish (Ubuntu Host Terminali):
Host tizimingizda buyruqlar satrini oching va quyidagi buyruqni bajaring:

```bash
ip -br addr show vmnet2
```
*Agar `vmnet2` da manzil belgilanmagan bo‘lsa, uni qo‘lda o‘rnating:*
```bash
sudo ip addr replace 10.130.10.1/24 dev vmnet2
sudo ip link set vmnet2 up
```

> **CLI Buyruqlari Tahlili:**
> - `ip`: Linux operatsion tizimida tarmoq interfeyslari, marshrutlash va manzillarni boshqaruvchi zamonaviy buyruq (eski `ifconfig` o‘rniga).
> - `-br` (`--brief`): Natijani juda ixcham, bir qatorda ko‘rsatish bayrog‘i.
> - `addr show vmnet2`: Faqat `vmnet2` nomli virtual tarmoq interfeysining IP manzillarini ekranga chiqarish.
> - `sudo`: Buyruqni tizim ma’muri (root) imtiyozlari bilan bajarish.
> - `ip addr replace 10.130.10.1/24 dev vmnet2`: `vmnet2` interfeysiga `10.130.10.1` IP manzilini va `255.255.255.0` (`/24`) tarmoq niqobini biriktirish. Agar avval boshqa IP bo‘lsa, uni almashtirish (`replace`).
> - `ip link set vmnet2 up`: Virtual tarmoq interfeysini faol (UP) holatga keltirish.

---

#### 1.3. WAF-SRV (Ubuntu Server VM) ichida paketlarni o‘rnatish:
*Eslatma: Paketlarni internetdan yuklab olish uchun virtual mashinaga vaqtincha ikkinchi adapter (NAT) ulangan bo‘lishi kerak yoki paketlarni o‘rnatib olgach NAT uziladi.*

Server konsolida quyidagi buyruqlarni bajaring:

```bash
sudo apt update
sudo apt install -y apache2 libapache2-mod-security2 modsecurity-crs python3 curl nano
sudo systemctl stop apache2
```

> **CLI Buyruqlari Tahlili:**
> - `sudo apt update`: Ubuntu repozitoriyalaridagi eng so‘nggi paketlar ro‘yxati va versiyalari haqidagi indeksni yangilaydi.
> - `sudo apt install`: Tizimga yangi dasturiy ta’minot paketlarini o‘rnatish buyrug‘i.
> - `-y` (`--yes`): O‘rnatish jarayonida disk hajmi va ruxsat so‘rovlariga avtomatik tarzda "Ha" (Yes) deb javob berish.
> - `apache2`: Dunyodagi eng barqaror HTTP veb-server dasturi. Ushbu laboratoriyada u teskari proksi (reverse proxy) va WAF platformasi rolini o‘ynaydi.
> - `libapache2-mod-security2`: Apache 2.4 veb-serveri uchun ModSecurity v2.x WAF dvigatelining rasmiy dinamik moduli.
> - `modsecurity-crs`: OWASP Core Rule Set — veb-ilovalar uchun keng tarqalgan kiberhujumlarni (XSS, SQLi, LFI, RCE) aniqlovchi tayyor qoidalar to‘plami.
> - `python3`: O‘quv backend serverining skriptini ishga tushirish uchun Python 3 interpretatori.
> - `curl`: Buyruqlar satri orqali HTTP/HTTPS so‘rovlar yuboruvchi qulay tarmoq mijozi.
> - `nano`: Konfiguratsiya fayllarini tahrirlash uchun qulay matn muharriri.
> - `sudo systemctl stop apache2`: O‘rnatilgach avtomatik ishga tushgan Apache serverini to‘xtatish (chunki uni keyingi bosqichlarda to‘g‘ri sozlab olib, keyin yoqamiz).

O‘rnatilgan paketlar versiyasini va CRS fayllarini tekshirish:
```bash
dpkg-query -W apache2 libapache2-mod-security2 modsecurity-crs
dpkg -L modsecurity-crs | grep -E 'crs-setup|rules/.*conf$'
```

> **CLI Buyruqlari Tahlili:**
> - `dpkg-query -W ...`: Ko‘rsatilgan paketlarning tizimda o‘rnatilgan aniq versiyalarini bir qatorda ko‘rsatadi (`-W` = `--show`).
> - `dpkg -L modsecurity-crs`: Paket tizimning qaysi papkalariga qanday fayllarni joylashtirganini to‘liq ro‘yxat qiladi (`-L` = `--listfiles`).
> - `|` (quvur / pipe): Birinchi buyruq natijasini ikkinchi buyruqqa kiruvchi ma’lumot sifatida uzatadi.
> - `grep -E 'crs-setup|rules/.*conf$'`: Ro‘yxat ichidan faqat kengaytirilgan regulyar ifoda (`-E`) bo‘yicha `crs-setup` so‘zi bor yoki `rules/` papkasidagi `.conf` bilan tugaydigan qoidalar fayllarini filtrlab ekranga chiqaradi.

---

#### 1.4. WAF-SRV da laboratoriya IP manzilini sozlash:
Serverda root huquqiga o‘ting va lab adapterini sozlang:

```bash
sudo -i
ip -br link
```
*Chiqgan ro‘yxatdan Host-Only tarmog‘iga ulangan interfeys nomini aniqlang (masalan: `ens33`, `ens37` yoki `enp0s8`). Shundan so‘ng:*

```bash
LAB_IF=ens33
ip link set "$LAB_IF" up
ip addr replace 10.130.10.10/24 dev "$LAB_IF"
ip -br addr
mkdir -p /opt/a13/evidence
```

> **CLI Buyruqlari Tahlili:**
> - `sudo -i`: Doimiy superuser (root) qobig‘iga (interactive root shell) o‘tish. Keyingi buyruqlarda har safar `sudo` yozish talab etilmaydi.
> - `ip -br link`: Tizimdagi barcha jismoniy va virtual tarmoq interfeyslarining nomlari hamda ularning MAC-manzillarini ko‘rish.
> - `LAB_IF=ens33`: Shell o‘zgaruvchisiga interfeys nomini saqlab qo‘yish (o‘zingizdagi haqiqiy nomga qarab o‘zgartiring).
> - `ip link set "$LAB_IF" up`: Tanlangan interfeysni yoqish.
> - `ip addr replace 10.130.10.10/24 dev "$LAB_IF"`: Ushbu interfeysga laboratoriyaning statik IP manzilini (`10.130.10.10`) biriktirish.
> - `mkdir -p /opt/a13/evidence`: Laboratoriya topshiriqlari fayllari va yig‘iladigan skrinshot/log dalillari uchun papkalar ierarxiyasini yaratish (`-p` ota papkalar bo‘lmasa ularni ham yaratadi).

---

### BOSQICH 2: O‘quv Backend Ilovasini Yaratish va Systemd Xizmati Qilish

Backend ilova mijoz so‘ragan ma’lumotlarni JSON ko‘rinishida aks ettiradi va `/redirect` manziliga so‘rov kelganda HTTP 302 qaytaradi.

#### 2.1. Backend Python skriptini yaratish (`/opt/a13/backend.py`):

```bash
cat > /opt/a13/backend.py <<'APP'
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlsplit, parse_qs
import json

class App(BaseHTTPRequestHandler):
    def answer(self, body=""):
        path = urlsplit(self.path)
        if path.path == "/redirect":
            self.send_response(302)
            self.send_header("Location", "http://127.0.0.1:8080/landing")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return
        data = {
            "service": "A13-backend",
            "path": path.path,
            "query": parse_qs(path.query),
            "body": body
        }
        raw = json.dumps(data, ensure_ascii=True).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("X-Content-Type-Options", "nosniff")
        self.send_header("Content-Length", str(len(raw)))
        self.end_headers()
        self.wfile.write(raw)

    def do_GET(self):
        self.answer()

    def do_POST(self):
        size = int(self.headers.get("Content-Length", "0"))
        if size > 131072:
            self.send_error(413)
            return
        self.answer(self.rfile.read(size).decode("utf-8", "replace"))

    def log_message(self, fmt, *args):
        case = self.headers.get("X-A13-Case", "-")
        print(case, fmt % args, flush=True)

ThreadingHTTPServer(("127.0.0.1", 8080), App).serve_forever()
APP
```

Huquqlarni to‘g‘rilash va sintaksisni kompilyatsiya qilib tekshirish:
```bash
chmod 755 /opt/a13
chmod 644 /opt/a13/backend.py
python3 -m py_compile /opt/a13/backend.py
```

> **CLI Buyruqlari Tahlili:**
> - `cat > /opt/a13/backend.py <<'APP' ... APP`: `Heredoc` usuli orqali ko‘p qatorli matnni faylga to‘g‘ridan-to‘g‘ri yozish. Tirnoq ichidagi `'APP'` belgilari fayl ichidagi o‘zgaruvchilar (`$`) bash tomonidan kengaytirib yuborilishining oldini oladi.
> - `chmod 755 /opt/a13`: `/opt/a13` papkasiga egasiga to‘liq (rwx), boshqa foydalanuvchilarga esa kirish va o‘qish (r-x) huquqini berish.
> - `chmod 644 /opt/a13/backend.py`: Python fayliga egasiga o‘qish-yozish (rw-), boshqalarga faqat o‘qish (r--) huquqini berish.
> - `python3 -m py_compile /opt/a13/backend.py`: Python skriptini bayt-kodga kompilyatsiya qilib, unda sintaksis xatolar yo‘qligini tekshirish (`.pyc` fayl hosil bo‘ladi).

---

#### 2.2. Backend ilovani Systemd servisi sifatida ro‘yxatdan o‘tkazish:
Ilova xavfsiz ishlashi uchun uni imtiyozsiz `www-data` foydalanuvchisi ostida va izolyatsiya qilingan muhitda ishga tushiramiz:

```bash
cat > /etc/systemd/system/a13-backend.service <<'UNIT'
[Unit]
Description=A13 teaching backend
After=network.target

[Service]
User=www-data
ExecStart=/usr/bin/python3 /opt/a13/backend.py
NoNewPrivileges=true
PrivateTmp=true
Restart=on-failure

[Install]
WantedBy=multi-user.target
UNIT
```

Servisni faollashtirish va ishga tushirish:
```bash
systemctl daemon-reload
systemctl enable --now a13-backend
curl --noproxy '*' --max-time 3 http://127.0.0.1:8080/health
ss -lntp 'sport = :8080'
```

> **CLI Buyruqlari Tahlili:**
> - `systemctl daemon-reload`: Systemd boshqaruvchisiga yangi yoki o‘zgartirilgan servis fayllarini (`*.service`) qayta o‘qishni buyuradi.
> - `systemctl enable --now a13-backend`: Servisni avtoulovga (enable) qo‘yadi va darhol (`--now`) ishga tushiradi.
> - `curl --noproxy '*' --max-time 3 http://127.0.0.1:8080/health`:
>   - `curl`: HTTP so‘rov yuborish.
>   - `--noproxy '*'`: Tizimda agar biron proxy o‘zgaruvchisi sozlangan bo‘lsa, ushbu manzil uchun uni e’tiborsiz qoldirish.
>   - `--max-time 3`: Agar server 3 soniya ichida javob bermasa, jarayonni to‘xtatish (osilib qolishdan saqlaydi).
>   - `http://127.0.0.1:8080/health`: Mahalliy backendning salomatlik holatini tekshirish. Kutilgan natija: `{"service": "A13-backend", "path": "/health", ...}` JSON javobi.
> - `ss -lntp 'sport = :8080'`:
>   - `ss` (Socket Statistics): Linuxdagi ochiq tarmoq ulanishlarini ko‘rsatuvchi eng tezkor vosita (eski `netstat` o‘rniga).
>   - `-l` (`--listening`): Faqat portni tinglab turgan rozetkalarni (sockets) ko‘rsatish.
>   - `-n` (`--numeric`): Xizmat nomlarini emas, sonli portlar va IP manzillarni ko‘rsatish.
>   - `-t` (`--tcp`): Faqat TCP trafigini filtrlash.
>   - `-p` (`--processes`): Portni qaysi dastur (PID va jarayon nomi) band qilganini ko‘rsatish.
>   - `'sport = :8080'`: Faqat manba porti (source port) 8080 bo‘lgan natijani filtrlash.
>   - **Kutilgan natija:** Tinglovchi manzil qat’iyan `127.0.0.1:8080` bo‘lishi shart! Agar `0.0.0.0:8080` yoki `*:8080` bo‘lsa, bu xavfsizlik talabining buzilishidir!

---

### BOSQICH 3: Apache Reverse Proxy-ni Sozlash

#### 3.1. Zaxira nusxa olish va Apache modullarini yoqish:
Har qanday muhim tizim o‘zgarishidan oldin zaxira olish shart:

```bash
cp -a /etc/apache2 /opt/a13/apache2-before
cp -a /etc/modsecurity /opt/a13/modsecurity-before
a2enmod proxy proxy_http headers unique_id security2
a2dissite 000-default
```

> **CLI Buyruqlari Tahlili:**
> - `cp -a ...`: Katalogning barcha fayllari, huquqlari va vaqt belgilarini o‘zgartirmasdan to‘liq arxiv nusxasini yaratish (`-a` = `--archive`).
> - `a2enmod` (Apache 2 Enable Module): Apache veb-serverida kerakli modullarni yoqish:
>   - `proxy`: Asosiy teskari proksi yadrosi.
>   - `proxy_http`: HTTP va HTTPS protokollari orqali so‘rovlarni backendga yo‘naltiruvchi qo‘shimcha modul.
>   - `headers`: HTTP so‘rov va javob sarlavhalarini o‘zgartirish, tozalash va yangi sarlavha qo‘shish moduli.
>   - `unique_id`: Har bir kiruvchi HTTP so‘rovi uchun takrorlanmas noyob 24 belgili identifikator (`UNIQUE_ID`) generatsiya qiluvchi modul. Bu WAF va kirish loglarini o‘zaro bog‘lashda asosiy kalit hisoblanadi.
>   - `security2`: ModSecurity WAF dvigateli moduli.
> - `a2dissite 000-default` (Apache 2 Disable Site): Standart o‘rnatilgan "Apache2 Ubuntu Default Page" sayt konfiguratsiyasini o‘chirib qo‘yish (port 80 ni bo‘shatish uchun).

---

#### 3.2. Apache tinglash porti va VirtualHost yaratish:
Apache faqat lab IP (`10.130.10.10:80`) da tinglashi kerak:

```bash
printf 'Listen 10.130.10.10:80\n' > /etc/apache2/ports.conf
```

A13 laboratoriyasi uchun yangi VirtualHost konfiguratsiyasini yozamiz:
```bash
cat > /etc/apache2/sites-available/a13.conf <<'VHOST'
<VirtualHost 10.130.10.10:80>
    ServerName a13.test
    ProxyRequests Off
    ProxyPreserveHost On

    ProxyPass / http://127.0.0.1:8080/ connectiontimeout=2 timeout=5
    ProxyPassReverse / http://127.0.0.1:8080/

    RequestHeader unset X-Forwarded-For
    RequestHeader unset X-Real-IP
    Header always set X-A13-Gateway "apache-waf"

    ErrorLog /var/log/apache2/a13-error.log
    LogFormat "%a %t \"%r\" %>s %{X-A13-Case}i %{UNIQUE_ID}e" a13
    CustomLog /var/log/apache2/a13-access.log a13
</VirtualHost>
VHOST
```

Saytni yoqish, sintaksisni tekshirish va Apache-ni qayta yuklash:
```bash
a2ensite a13
apache2ctl configtest
systemctl restart apache2
apache2ctl -M | grep -E 'proxy|security2|unique_id|headers'
apache2ctl -S
```

> **CLI Buyruqlari Tahlili:**
> - `printf 'Listen 10.130.10.10:80\n' > /etc/apache2/ports.conf`: Apache standart `Listen 80` (barcha interfeyslarda tinglash) sozlamasini o‘chirib, faqatgina bizning laboratoriya IP manzilimizda tinglashga majburlash.
> - `a2ensite a13` (Apache 2 Enable Site): `/etc/apache2/sites-available/a13.conf` fayliga `/etc/apache2/sites-enabled/` katalogidan simvolik havola (symlink) yaratib, saytni faollashtirish.
> - `apache2ctl configtest`: Barcha konfiguratsiya fayllarining sintaksisini tekshiradi. **"Syntax OK"** javobi chiqishi shart! Agar xatolik bo‘lsa, qaysi qatorda xato ketgani aniq ko‘rsatiladi.
> - `systemctl restart apache2`: Apache xizmatini to‘liq qayta ishga tushirish.
> - `apache2ctl -M`: Apache xotirasiga yuklangan barcha faol modullar ro‘yxatini chiqaradi (`-M` = list modules).
> - `apache2ctl -S`: Virtual xostlar xaritasini chiqaradi (qaysi IP va port qaysi VirtualHost fayliga biriktirilganini ko‘rsatadi).

---

#### 3.3. Reverse Proxy faoliyatini Ubuntu Hostdan tekshirish (Dastlabki sinov):
Endi asosiy **Ubuntu Host** terminaliga o‘ting (yoki mijoz konsolida) va Apache reverse proxy orqali backendga yetib borilayotganini tekshiring:

```bash
# Ubuntu Host Terminali:
export U='http://10.130.10.10'
curl --noproxy "*" --max-time 5 -i "$U/health"
curl --noproxy "*" --max-time 5 -i "$U/redirect"
```

> **Natijalar Tahlili:**
> - Birinchi so‘rov (`/health`): Serverdan `HTTP/1.1 200 OK` javobi va `X-A13-Gateway: apache-waf` sarlavhasi qaytadi. Tana qismida backend JSON ma’lumoti keladi. Bu proksi backend bilan to‘g‘ri bog‘langanini isbotlaydi.
> - Ikkinchi so‘rov (`/redirect`): Serverdan `HTTP/1.1 302 Found` javobi qaytadi. Eng muhimi: `Location:` sarlavhasida ichki `http://127.0.0.1:8080/landing` emas, balki `ProxyPassReverse` tomonidan qayta yozilgan tashqi manzil — `http://10.130.10.10/landing` aks etishi kerak!

---

### BOSQICH 4: ModSecurity va OWASP CRS-ni Integratsiya Qilish

#### 4.1. Konfiguratsiya fayllarini tayyorlash (WAF-SRV VM konsoli):
Tizimda tavsiya etilgan ModSecurity konfiguratsiyasini nusxalash va boshlang‘ich holatda **DetectionOnly** rejimiga sozlash:

```bash
test -f /etc/modsecurity/modsecurity.conf-recommended
test -f /etc/modsecurity/crs/crs-setup.conf
ls /usr/share/modsecurity-crs/rules/*.conf | head -n 5

cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
sed -i 's/^SecRuleEngine .*/SecRuleEngine DetectionOnly/' /etc/modsecurity/modsecurity.conf

# Bo'sh runtime istisnolar va mahalliy qoidalar fayllarini yaratish:
printf '# A13 runtime exclusions\n' > /etc/modsecurity/a13-before.conf
printf '# A13 local rules\n' > /etc/modsecurity/a13-local.conf
```

> **CLI Buyruqlari Tahlili:**
> - `test -f <fayl>`: Fayl haqiqatan mavjudligini tekshiradi (agar mavjud bo‘lsa exit code 0 qaytaradi).
> - `cp ...`: Standart tavsiya etilgan shablon faylni ishchi `modsecurity.conf` sifatida nusxalash.
> - `sed -i 's/^SecRuleEngine .*/SecRuleEngine DetectionOnly/' ...`: `sed` oqimli muharriri orqali fayl ichidagi `SecRuleEngine` bilan boshlanuvchi qatorni topib, uning qiymatini darhol joyida (`-i` = in-place) `DetectionOnly` ga o‘zgartirish.

---

#### 4.2. Qoidalarning qat’iy yuklanish zanjirini sozlash (`security2.conf`):
ModSecurity moduli konfiguratsiyasini ochib, qoidalarning to‘g‘ri tartibda yuklanishini ta’minlaymiz:

```bash
cat > /etc/apache2/mods-available/security2.conf <<'WAF'
<IfModule security2_module>
    SecDataDir /var/cache/modsecurity
    Include /etc/modsecurity/modsecurity.conf

    SecAuditEngine RelevantOnly
    SecAuditLogType Serial
    SecAuditLog /var/log/apache2/a13-audit.log
    SecAuditLogParts ABFHZ

    Include /etc/modsecurity/a13-before.conf
    Include /etc/modsecurity/crs/crs-setup.conf
    Include /usr/share/modsecurity-crs/rules/*.conf
    Include /etc/modsecurity/a13-local.conf
</IfModule>
WAF
```

Konfiguratsiyani tekshirish va yuklanish zanjirini hujjatlashtirish:
```bash
apache2ctl configtest
apache2ctl -t -D DUMP_INCLUDES > /opt/a13/evidence/includes.txt
grep -E 'modsecurity|crs' /opt/a13/evidence/includes.txt
systemctl reload apache2
```

> **CLI Buyruqlari Tahlili:**
> - `apache2ctl -t -D DUMP_INCLUDES`: Apache serveri qaysi konfiguratsiya fayllarini qanday ketma-ketlikda xotiraga yuklaganini to‘liq daraxt ko‘rinishida chiqaradi.
> - `grep -E 'modsecurity|crs' ...`: Yuklangan fayllar ichidan ModSecurity va CRS qoidalari to‘g‘ri tartibda joylashganini tasdiqlash uchun filtrlaydi.
> - `systemctl reload apache2`: Apache-ni to‘xtatmasdan (uptime uzilmasdan), konfiguratsiya o‘zgarishlarini muloyimlik bilan qayta yuklash.

Tekshiruv: `modsecurity.conf` ichida POST so‘rov tanasini tekshirish yoqilganligiga ishonch hosil qiling:
```bash
grep -E '^SecRuleEngine|^SecRequestBodyAccess' /etc/modsecurity/modsecurity.conf
```
*Chiqishda `SecRuleEngine DetectionOnly` va `SecRequestBodyAccess On` ko‘rinishi shart!*

---

### BOSQICH 5: Mahalliy Qoidalar va Kuzatuv (DetectionOnly) Rejimida Sinash (T1–T3)

#### 5.1. O‘quv qoidalarini qo‘shish (`/etc/modsecurity/a13-local.conf`):
Biz ikkita maxsus o‘quv qoidasini yaratamiz:
1. `Rule 1001300`: Phase 1 da ishlaydi. Agar so‘rov sarlavhalarida `X-A13-Test: block-me` bo‘lsa, uni bloklash uchun belgilaydi.
2. `Rule 1001301`: Phase 2 da ishlaydi. Agar forma parametrlarida (`ARGS:comment`) "script" so‘zi uchrasa, uni rad etish uchun belgilaydi (bu qoida ataylab o‘ta sodda va qopol yozilgan, keyinchalik false positive keltirib chiqarish uchun ishlatiladi).

```bash
cat > /etc/modsecurity/a13-local.conf <<'RULES'
SecRule REQUEST_HEADERS:X-A13-Test "@streq block-me" \
    "id:1001300,phase:1,t:none,deny,status:403,log,msg:'A13 marker'"

SecRule ARGS:comment "@contains script" \
    "id:1001301,phase:2,t:none,t:lowercase,deny,status:403,log,msg:'A13 broad comment rule'"
RULES

apache2ctl configtest && systemctl reload apache2
```

> **Qoidalar Sintaksisi Tahlili:**
> - `SecRule REQUEST_HEADERS:X-A13-Test`: Tekshiriladigan o‘zgaruvchi — kiruvchi HTTP sarlavhalari orasidagi `X-A13-Test`.
> - `@streq block-me`: Operator — satr aynan `block-me` ga teng ekanligini tekshiradi (string equal).
> - `id:1001300`: Qoidaning noyob identifikatori.
> - `phase:1`: Qoida 1-bosqichda (sarlavhalar o‘qilganda) bajariladi.
> - `t:none`: Ma’lumotga hech qanday dastlabki transformatsiya qo‘llanmaydi.
> - `deny,status:403`: Agar qoida tetiklansa, so‘rovni rad etish va HTTP 403 statusini qaytarish amali.
> - `log,msg:'...'`: Qoida ishlaganda jurnalga yoziladigan xabar.
> - Ikkinchi qoidada: `ARGS:comment` — POST yoki GET orqali kelgan `comment` parametri; `@contains script` — matn ichida "script" so‘zi bormi-yo‘qligini tekshirish; `t:lowercase` — tekshirishdan oldin matnni to‘liq kichik harflarga o‘girish.

---

#### 5.2. T1–T3 Sinovlarini Bajarish (Ubuntu Host Terminali):
*Eslatma: Hozirda WAF `DetectionOnly` rejimida! Demak, qoidalar xabar bersa ham, barcha so‘rovlar o‘tib ketishi va HTTP 200 qaytishi kerak!*

Ubuntu Host terminalida buyruqlarni bajaring:

```bash
# Ubuntu Host Terminali:
export U='http://10.130.10.10'

# T1: Oddiy qonuniy so'rov (Hech qanday qoidani buzmaydi)
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T1" "$U/health"

# T2: Mahalliy qoida markeri (X-A13-Test: block-me sarlavhasi bilan)
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T2" -H "X-A13-Test: block-me" "$U/health"

# T3: OWASP CRS uchun XSS belgili so'rov (?q=<script>alert(1)</script>)
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T3" -G --data-urlencode "q=<script>alert(1)</script>" "$U/search"
```

> **CLI Buyruqlari Tahlili:**
> - `-H "X-A13-Case: T1"`: So‘rovga maxsus sarlavha qo‘shadi. Bu test nomini Apache access log, error log, audit log va backend jurnalida oson qidirib topish va filtr qilish uchun kerak.
> - `-G` (`--get`): `--data` yoki `--data-urlencode` orqali berilgan parametrlarni POST emas, balki URL manziliga GET so‘rovi query stringi sifatida biriktiradi (`/search?q=...`).
> - `--data-urlencode "q=<script>alert(1)</script>"`: Maxsus belgilarni (`<`, `>`, `/`) brauzer kabi to‘g‘ri URL-kodlashdan (URL encode: `%3Cscript%3E...`) o‘tkazadi.

**Kutilgan natija:**
- T1: HTTP 200 OK
- T2: HTTP 200 OK (Kuzatuv rejimida bo‘lgani uchun bloklanmadi!)
- T3: HTTP 200 OK (Kuzatuv rejimida bo‘lgani uchun bloklanmadi!)

Endi WAF-SRV serverida jurnallarni tekshirib, qoidalar haqiqatan ishlaganini isbotlaymiz (WAF-SRV VM konsoli):
```bash
grep -E '1001300|941[0-9]{3}|949110' /var/log/apache2/a13-error.log
journalctl -u a13-backend --since '15 minutes ago' --no-pager | grep -E 'T1|T2|T3'
```
*Tahlil: Error logda T2 uchun `1001300`, T3 uchun esa CRSning `941xxx` (XSS aniqlash) qoidalari tetiklangani qayd etilgan, lekin backend jurnalida T1, T2 va T3 so‘rovlarining barchasi backendga yetib borgani tasdiqlanadi!*

---

### BOSQICH 6: Faol Bloklash (On) Rejimiga O‘tish va Dalillarni Bog‘lash (T4–T6)

#### 6.1. WAFni faol bloklash rejimiga o‘tkazish (WAF-SRV VM konsoli):

```bash
cp /etc/modsecurity/modsecurity.conf /opt/a13/modsecurity-detect.conf
sed -i 's/^SecRuleEngine .*/SecRuleEngine On/' /etc/modsecurity/modsecurity.conf
apache2ctl configtest && systemctl reload apache2
grep '^SecRuleEngine' /etc/modsecurity/modsecurity.conf
```
*Natija: `SecRuleEngine On` chiqishi shart! Endi WAF qoidani buzgan har qanday so‘rovni shafqatsizlarcha bloklaydi.*

---

#### 6.2. T4–T6 Sinovlarini Bajarish (Ubuntu Host Terminali):

```bash
# Ubuntu Host Terminali:
export U='http://10.130.10.10'

# T4: Oddiy qonuniy so'rov (Hali ham ishlashi kerak)
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T4" "$U/health"

# T5: Mahalliy qoida markeri (Endi bloklanishi shart!)
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T5" -H "X-A13-Test: block-me" "$U/health"

# T6: OWASP CRS aniqlagan XSS hujumi (Endi bloklanishi shart!)
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T6" -G --data-urlencode "q=<script>alert(1)</script>" "$U/search"
```

**Kutilgan natija:**
- **T4:** `HTTP/1.1 200 OK` — qonuniy trafikka WAF xalaqit bermaydi.
- **T5:** `HTTP/1.1 403 Forbidden` — Rule 1001300 ishga tushib, so‘rovni to‘xtatdi.
- **T6:** `HTTP/1.1 403 Forbidden` — OWASP CRS qoidalari anomaliya balini oshirdi va Rule 949110 so‘rovni blokladi!

---

#### 6.3. Jurnallarni tahlil qilish va Tranzaksiyalarni o‘zaro bog‘lash (WAF-SRV VM):
WAF-SRV serverida dalillarni tekshiramiz:

```bash
tail -n 10 /var/log/apache2/a13-access.log
grep -E '1001300|941[0-9]{3}|949110' /var/log/apache2/a13-error.log | tail -n 10
journalctl -u a13-backend --since '15 minutes ago' --no-pager | grep -E 'T4|T5|T6'
```

> **Loglar Tahlili va Korrelyatsiya Zanjiri:**
> 1. `a13-access.log` faylida T4 uchun `200`, T5 va T6 uchun `403` qayd etilgan. Har bir qator oxirida o‘sha so‘rovning noyob **`UNIQUE_ID`** kodi (masalan, `Z_1a2b3c4d5e...`) mavjud.
> 2. `a13-error.log` faylida aynan shu `UNIQUE_ID` bilan bog‘langan xatoliklar ko‘rinadi:
>    - T5 da: `[id "1001300"] [msg "A13 marker"] [data "block-me"]`
>    - T6 da: Dastlab `[id "941100"]` (XSS Attack Detected), so‘ngra `[id "949110"] [msg "Inbound Anomaly Score Exceeded (Total Score: 5)"]`
> 3. `a13-backend` jurnalida esa: T4 so‘rovi backendga yetib borgan, lekin **T5 va T6 so‘rovlari backendga umuman yetib bormagan!** Bu WAF Apache darajasidayoq hujumni to‘xtatib, ichki serverni yuklama va xavfdan asrab qolganini 100% isbotlaydi.

---

### BOSQICH 7: False Positive Holati va Tor Runtime Istisno Yaratish (T7–T10)

#### 7.1. T7: Qonuniy biznes so‘rovi xato bloklanishi (Ubuntu Host Terminali):
Foydalanuvchi o‘quv portalining `/submit` manziliga qonuniy izoh yuboradi: `"comment=JavaScript darsi"`.

```bash
# Ubuntu Host Terminali:
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T7" \
    --data-urlencode "comment=JavaScript darsi" "$U/submit"
```

**Kutilgan natija:**
- Serverdan `HTTP/1.1 403 Forbidden` qaytadi!
- Serverdagi `a13-error.log` ni tekshirsak:
  `ModSecurity: Access denied with code 403 ... [id "1001301"] [msg "A13 broad comment rule"] [data "Matched Data: script found within ARGS:comment: javascript darsi"]`
- **Xulosa (False Positive):** Foydalanuvchi hech qanday buzg‘unchilik qilgani yo‘q! U shunchaki "JavaScript darsi" deb yozdi. Ammo bizning 1001301 qoidamiz o‘ta sodda bo‘lgani uchun "JavaScript" so‘zining ichida "script" bo‘lagi borligini ko‘rib, uni xakerlik deb baholadi va blokladi. Bu klassik **False Positive (soxta ijobiy)** holatidir!

---

#### 7.2. Tor Runtime Istisno (Exclusion Rule) yaratish (WAF-SRV VM):
Muammoni qanday yechamiz? Butun WAFni o‘chiramizmi? Yo‘q! 1001301 qoidasini butunlay o‘chirib tashlaymizmi? Yo‘q!
Biz faqatgina:
1. So‘rov aynan `/submit` URI manziliga kelganda;
2. HTTP metodi aynan `POST` bo‘lganda;
3. Faqatgina `1001301` identifikatorli qoidadan `ARGS:comment` parametrini chiqarib tashlaymiz (`ctl:ruleRemoveTargetById=1001301;ARGS:comment`).

Buni amalga oshirish uchun `/etc/modsecurity/a13-before.conf` fayliga qoidalar zanjirini (Chained rule) yozamiz:

```bash
cp /etc/modsecurity/a13-before.conf /opt/a13/exclusions-before.conf

cat > /etc/modsecurity/a13-before.conf <<'EXCLUDE'
SecRule REQUEST_URI "@streq /submit" \
    "id:1001302,phase:1,t:none,pass,nolog,chain"
SecRule REQUEST_METHOD "@streq POST" \
    "t:none,ctl:ruleRemoveTargetById=1001301;ARGS:comment"
EXCLUDE

apache2ctl configtest && systemctl reload apache2
```

> **Istisno Qoidasi Tahlili:**
> - `SecRule REQUEST_URI "@streq /submit"`: 1-faza boshida so‘rov URI manzili aynan `/submit` ga tengligi tekshiriladi (`@streq`).
> - `chain`: Zanjirli qoida. Keyingi `SecRule` ham qanoatlantirilsagina amal bajariladi.
> - `SecRule REQUEST_METHOD "@streq POST"`: So‘rov usuli aynan `POST` bo‘lishi shart.
> - `ctl:ruleRemoveTargetById=1001301;ARGS:comment`: ModSecurity boshqaruv amali (`ctl`). U faqatgina ushbu tranzaksiya doirasida 1001301-qoidaning tekshiruv ro‘yxatidan `ARGS:comment` nishonini o‘chirib turadi.
> - Nega `a13-before.conf` da? Chunki bu fayl `security2.conf` ichida qoidalardan oldin yuklanadi va Phase 1 da ishlaydi. 1001301 qoidasi esa Phase 2 da ishlaydi. Demak, istisno o‘z vaqtida nishonni olib tashlashga ulguradi!

---

#### 7.3. T8–T10 Sinovlari: Tuzatish va Xavfsizlik Saqlanganligini Isbotlash (Ubuntu Host Terminali):

Endi Ubuntu Host terminalida 3 ta muhim nazorat sinovini bajaramiz:

```bash
# Ubuntu Host Terminali:
export U='http://10.130.10.10'

# T8: Aynan o'sha ruxsat berilgan qonuniy matn (/submit ga)
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T8" \
    --data-urlencode "comment=JavaScript darsi" "$U/submit"

# T9: Xuddi shu matn BOSHQA endpointga yuborilganda (/other ga)
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T9" \
    --data-urlencode "comment=JavaScript darsi" "$U/other"

# T10: Aynan /submit ga haqiqiy XSS hujumi yuborilganda
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T10" \
    --data-urlencode "comment=<script>alert(1)</script>" "$U/submit"
```

> **Natijalar Tahlili va Himoya Isboti:**
> - **T8 Natijasi:** `HTTP/1.1 200 OK`! Qonuniy foydalanuvchining so‘rovi muvaffaqiyatli o‘tdi va backendga yetib bordi. False positive bartaraf etildi!
> - **T9 Natijasi:** `HTTP/1.1 403 Forbidden`! Nega? Chunki so‘rov `/other` manziliga yuborildi. Bizning istisnomiz faqat `/submit` uchun amal qiladi. Demak, istisno juda tor doirada ishlashi va boshqa joylarga ta’sir qilmasligi isbotlandi (Rule 1001301 saqlangan).
> - **T10 Natijasi:** `HTTP/1.1 403 Forbidden`! Eng muhim nuqta: Nega 403 berdi? Axir `/submit` da comment maydonidan 1001301 olib tashlangan ediku? Ha, lekin biz OWASP CRS qoidalarini o‘chirmagan edik! OWASP CRS ning 941xxx qoidalari va 949110 anomaliya baholovchisi ushbu kiritmadagi `<script>alert(1)</script>` matnini aniqlab, so‘rovni darhol blokladi!
> - **Xulosa:** Biz qonuniy so‘rovga yo‘l ochdik, ammo tizimda kiberhujumlardan himoyani zarracha ham susaytirmadik!

---

### BOSQICH 8: Origin Yashirilishi va Redirect Sarlavhalarini Tekshirish (T11–T12)

#### 8.1. T11–T12 Sinovlari (Ubuntu Host Terminali):

```bash
# Ubuntu Host Terminali:

# T11: Backendning ichki 8080 portiga tashqaridan to'g'ridan-to'g'ri ulanib ko'rish
nc -zv 10.130.10.10 8080

# T12: Backend redirect (302) berganda sarlavha to'g'ri qayta yozilishi
curl --noproxy "*" --max-time 5 -i -H "X-A13-Case: T12" "$U/redirect"
```

> **CLI Buyruqlari Tahlili:**
> - `nc` (Netcat): Tarmoq orqali TCP va UDP paketlar yuboruvchi va portlarni tekshiruvchi "shveysariya pichoqchasi" vositasi (Linuxda Windowsning `Test-NetConnection` o‘rniga ishlatiladi).
> - `-z` (`--zero-I/O`): Ma’lumot yubormasdan faqat port ochiq yoki yopiqligini skaner qilish.
> - `-v` (`--verbose`): Natijani batafsil ekranga chiqarish.
> - **T11 Kutilgan Natijasi:** `nc: connect to 10.130.10.10 port 8080 (tcp) failed: Connection refused`. Backend porti tashqi interfeysda butunlay yopiq! Chunki u faqat loopback (`127.0.0.1:8080`) da ishlaydi.
> - **T12 Kutilgan Natijasi:** Serverdan `HTTP/1.1 302 Found` qaytadi va uning ichidagi sarlavha: `Location: http://10.130.10.10/landing` bo‘ladi. `ProxyPassReverse` ichki loopback manzilini mijozdan yashirib, proksi manziliga aylantirdi.

---

### BOSQICH 9: Barcha 12 ta Sinov Natijalarining Umumiy Jadvali

Quyidagi jadval amaliy mashg‘ulot davomida bajarilgan 12 ta sinovning to‘liq solishtirma dalilini o‘z ichiga oladi:

| Test ID | WAF Holati va Kiritma Tavsifi | Kutilgan Status | Amaldagi Status | Qoida ID va Asosiy Dalil | Backendga Yetganmi? |
| :---: | :--- | :---: | :---: | :--- | :---: |
| **T1** | `DetectionOnly`; Oddiy GET `/health` | **200 OK** | 200 OK | Hech qanday qoida ishlamadi; toza trafik | Ha, JSON qaytdi |
| **T2** | `DetectionOnly`; `X-A13-Test: block-me` | **200 OK** | 200 OK | Error logda Rule `1001300` qayd etildi, lekin bloklanmadi | Ha, JSON qaytdi |
| **T3** | `DetectionOnly`; `?q=<script>alert(1)</script>` | **200 OK** | 200 OK | CRS `941xxx` anomaliya ballari yozildi, ammo bloklanmadi | Ha, JSON qaytdi |
| **T4** | `SecRuleEngine On`; Oddiy GET `/health` | **200 OK** | 200 OK | WAF qonuniy so‘rovni to‘siqsiz o‘tkazdi | Ha, JSON qaytdi |
| **T5** | `SecRuleEngine On`; `X-A13-Test: block-me` | **403 Forbidden** | 403 Forbidden | Rule `1001300` tomonidan to‘xtatildi | **Yo‘q**, bloklandi |
| **T6** | `SecRuleEngine On`; `?q=<script>alert(1)</script>` | **403 Forbidden** | 403 Forbidden | CRS `941xxx` (XSS aniqlash) va `949110` (Inbound Anomaly Score Exceeded) | **Yo‘q**, bloklandi |
| **T7** | `SecRuleEngine On`; POST `/submit` ("JavaScript darsi") | **403 Forbidden** | 403 Forbidden | Rule `1001301` xato ishladi (**False Positive!**) | **Yo‘q**, xato bloklandi |
| **T8** | Tor istisnodan keyin; POST `/submit` ("JavaScript darsi") | **200 OK** | 200 OK | Rule `1001302` istisnosi `1001301` ni olib tashladi | Ha, JSON qaytdi |
| **T9** | Tor istisnodan keyin; POST `/other` ("JavaScript darsi") | **403 Forbidden** | 403 Forbidden | Rule `1001301` o‘z kuchida (istisno faqat `/submit` uchun) | **Yo‘q**, bloklandi |
| **T10** | Tor istisnodan keyin; POST `/submit` ("`<script>alert(1)</script>`") | **403 Forbidden** | 403 Forbidden | CRS `941xxx` va `949110` XSS himoyasi to‘liq saqlangan! | **Yo‘q**, bloklandi |
| **T11** | Backend TCP 8080 portiga to‘g‘ridan-to‘g‘ri ulanish | **Ulanmaydi** | Connection refused | `ss -lntp` da faqat `127.0.0.1:8080`; tashqaridan yopiq | Ulanish yo‘q |
| **T12** | GET `/redirect` so‘rovi | **302 Found** | 302 Found | `ProxyPassReverse` tufayli `Location: http://10.130.10.10/landing` | Ha, proksi orqali |

---

### BOSQICH 10: Bloklangan Tranzaksiya Kartasi (Real Dalil Namunasi)

Laboratoriya hisobotini topshirish uchun T6 sinovi bo‘yicha to‘ldirilgan tranzaksiya kartasi:

```yaml
Tranzaksiya Kartasi (Audit Evidence Card):
  Test ID: T6
  Sana va Vaqt: 2026-11-04 18:45:12 UTC+5 (Toshkent vaqti)
  Mijoz Manba IP: 10.130.10.1 (Ubuntu Host vmnet2 interfeysi)
  Server IP va Port: 10.130.10.10:80 (Apache 2.4 WAF)
  HTTP Usuli va URI: GET /search?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E
  Access Log UNIQUE_ID: "Z8g7s38AAQEAAH2W6dMAAAAB"
  
  Aniqlash Qoidasi (Detection Rules):
    - Qoida ID: 941100 (OWASP CRS - XSS Filter - Category 1: Script Tag Vector)
    - Xabar: "XSS Attack Detected via libinjection"
    - Anomaliya Bali: Inbound Anomaly Score +5
    
  Yakuniy Bloklash Qoidasi (Blocking Rule):
    - Qoida ID: 949110 (Inbound Anomaly Score Exceeded)
    - Amal: deny, status 403
    - Xabar: "Inbound Anomaly Score Exceeded (Total Score: 5, Threshold: 5)"
    
  Natija:
    - Mijozga qaytgan status: HTTP 403 Forbidden
    - Backendga yetganmi: YO'Q (a13-backend journalida T6 qayd etilmadi)
```

---

### BOSQICH 11: Laboratoriya Dalillarini Yig‘ish va Arxivlash

Barcha konfiguratsiya fayllari va loglarni hisobot uchun yagona `/opt/a13/evidence/` papkasiga yig‘ish (WAF-SRV VM konsoli):

```bash
cp /etc/apache2/sites-available/a13.conf /opt/a13/evidence/
cp /etc/apache2/mods-available/security2.conf /opt/a13/evidence/
cp /etc/modsecurity/modsecurity.conf /opt/a13/evidence/
cp /etc/modsecurity/a13-*.conf /opt/a13/evidence/
cp /opt/a13/backend.py /opt/a13/evidence/
cp /etc/systemd/system/a13-backend.service /opt/a13/evidence/
cp /var/log/apache2/a13-*.log /opt/a13/evidence/
journalctl -u a13-backend --since '90 minutes ago' --no-pager > /opt/a13/evidence/backend-journal.txt
dpkg-query -W apache2 libapache2-mod-security2 modsecurity-crs > /opt/a13/evidence/versions.txt

# Dalillar arxivini yaratish:
tar -czvf /opt/a13/a13_evidence.tar.gz -C /opt/a13/evidence .
```

> **CLI Buyruqlari Tahlili:**
> - `tar -czvf /opt/a13/a13_evidence.tar.gz -C /opt/a13/evidence .`:
>   - `tar`: Fayllarni bitta arxivga birlashtirish (Tape Archive).
>   - `-c` (`--create`): Yangi arxiv fayl yaratish.
>   - `-z` (`--gzip`): Arxivni gzip algoritmi yordamida siqish (hajmni kichraytirish).
>   - `-v` (`--verbose`): Arxivlanayotgan har bir faylni ekranda ko‘rsatish.
>   - `-f`: Yaratiladigan arxiv faylining nomi.
>   - `-C /opt/a13/evidence .`: Arxivga kirmasdan oldin o‘sha katalogga o‘tish va faqat ichidagi fayllarni siqish (ortiqcha ota papkalar yo‘lini saqlamaslik uchun).

---

## 4. NOSOZLIKLARNI BARTARAF ETISH (TROUBLESHOOTING) VA NAZORAT SAVOLLARI

### 4.1. Nosozliklarni Aniqlash va Tuzatish Jadvali

| Xatolik Belgisi | Ehtimoliy Sabab | Qanday Tekshirish va Tuzatish Kerak? |
| :--- | :--- | :--- |
| **Apache ishga tushmayapti (`failed`)** | Port band yoki konfiguratsiyada sintaksis xatosi bor | `apache2ctl configtest` buyrug‘ini bering. Xatoli fayl va qatorni toping. `ss -lntp 'sport = :80'` orqali portni boshqa xizmat band qilmaganini tekshiring. |
| **"Duplicate rule ID" xatosi** | Bir xil qoida IDsi ikki marta yuklangan | `apache2ctl -t -D DUMP_INCLUDES` qilib, `security2.conf` ichida bir xil qoidalar yoki eski fayllar ikki marta `Include` qilinmaganiga ishonch hosil qiling. |
| **Marker so‘rovi (T5) 200 qaytarmoqda (`On` yozilgan)** | `modsecurity.conf` o‘zgargach Apache qayta yuklanmagan | `grep '^SecRuleEngine' /etc/modsecurity/modsecurity.conf` ni tekshiring va `systemctl restart apache2` buyrug‘ini bering. |
| **Marker 403 beryapti, ammo XSS (T6) 200 olyapti** | OWASP CRS qoidalari yuklanmagan | `security2.conf` ichida `Include /usr/share/modsecurity-crs/rules/*.conf` qatori borligini va bu yo‘lda fayllar mavjudligini tekshiring. |
| **POST so‘rovlari WAF tomonidan tekshirilmayapti** | So‘rov tanasini o‘qish o‘chirilgan | `modsecurity.conf` ichida `SecRequestBodyAccess On` ekanini tekshiring. |
| **T8 sinovida ham 403 Forbidden olinmoqda** | Istisno qoidasi noto‘g‘ri yozilgan yoki joyi xato | `a13-before.conf` fayli `security2.conf` ichida CRS va mahalliy qoidalardan OLDIN turganini, URI (`/submit`) va HTTP method (`POST`) to‘g‘ri yozilganini tekshiring. |
| **T10 sinovi 200 qaytarmoqda (Xavfli!)** | Istisno o‘ta keng yozilgan, butun WAF yoki CRS o‘chirilgan | Istisnoda faqat `ctl:ruleRemoveTargetById=1001301;ARGS:comment` bo‘lishi kerak. `SecRuleEngine Off` yoki `ruleRemoveById` butun qoidani o‘chirib yubormasligi lozim! |
| **HTTP 502 Bad Gateway yoki 503 Service Unavailable** | Python backend ishlamayapti | `systemctl status a13-backend` ni tekshiring. `curl http://127.0.0.1:8080/health` orqali to‘g‘ridan-to‘g‘ri ulanib ko‘ring. |

---

### 4.2. Nazorat Savollari va Ularning Mukammal Javoblari

1. **Forward va Reverse Proxy-ning asosiy farqi nimada? Nega bu laboratoriyada `ProxyRequests On` kerak emas?**
   - *Javob:* Forward proxy mijozlar tomonida turadi va ularning ixtiyoriy tashqi internet resurslariga chiqishiga vositachilik qiladi. Reverse proxy esa serverlar oldida turadi va tashqi mijozlarning aniq belgilangan ichki backend servislariga xavfsiz ulanishini ta’minlaydi. `ProxyRequests On` Apache-ni ochiq forward proxy rejimiga o‘tkazadi; bizga bu kerak emas, chunki biz tashqi dunyoga ochiq tranzit emas, balki faqat o‘zimizning ichki backendimizni himoyalovchi Reverse Proxy (`ProxyRequests Off` va `ProxyPass`) quramiz.

2. **`ProxyPassReverse` qaysi javob qismini tuzatadi? HTML sahifa ichidagi barcha URLlarni o‘zgartiradimi?**
   - *Javob:* `ProxyPassReverse` faqat HTTP javob sarlavhalarini — asosan `Location`, `Content-Location` va `URI` sarlavhalarini qayta yozadi. Agar backend 301/302 Redirect qaytarsa, u yerdagi ichki manzilni (`http://127.0.0.1:8080/`) tashqi proksi manziliga (`http://10.130.10.10/`) moslashtiradi. U HTML sahifaning tana (body) qismidagi havolalarni avtomatik o‘zgartirmaydi (HTML ichini o‘zgartirish uchun `mod_substitute` yoki `mod_proxy_html` talab qilinadi).

3. **`DetectionOnly` holatida WAF xabari chiqib, so‘rov 200 olishi nimani anglatadi?**
   - *Javob:* Bu ModSecurity dvigateli so‘rovni to‘liq tahlil qilganini, hujum shablonini topganini, anomaliya ballarini hisoblaganini va error/audit jurnaliga yozganini, ammo passiv (kuzatuv) rejimida bo‘lgani sababli `deny` (bloklash) amalini bajarmay, so‘rovni backendga o‘tkazib yuborganini bildiradi.

4. **Mahalliy `1001300` qoidasining ishlashi nega CRS ishlayotganini isbotlamaydi?**
   - *Javob:* Chunki 1001300 qoidasi mahalliy faylda (`a13-local.conf`) yozilgan bitta oddiy qoida bo‘lib, u faqat ModSecurity dvigateli faolligini ko‘rsatadi. OWASP CRS esa `/usr/share/modsecurity-crs/rules/` katalogidagi yuzlab murakkab qoidalardan iborat alohida to‘plamdir. Agar CRS konfiguratsiyasi ulanmagan bo‘lsa ham, 1001300 ishlayveradi. Shuning uchun CRS faoliyatini alohida XSS/SQLi testlari (T3/T6) orqali tekshirish shart.

5. **`941xxx` aniqlash qoidasi bilan `949110` bloklash tekshiruvi qanday farqlanadi?**
   - *Javob:* `941xxx` qoidalari bevosita tahdid imzolarini (masalan, XSS teglari, JavaScript funksiyalarini) aniqlaydi va anomaliya ballarini oshiradi (`TX:anomaly_score +5`), ammo so‘rovni darhol bloklamaydi. `949110` qoidasi esa Phase 2 ning eng oxirida ishga tushib, barcha qoidalar yig‘gan umumiy anomaliya balini tekshiradi va agar u belgilangan chegaradan oshgan bo‘lsa, yakuniy bloklash amalini (`deny`, status 403) qo‘llaydi.

6. **POST tanasini WAF tahlil qilishi uchun qaysi asosiy sozlama kerak?**
   - *Javob:* `modsecurity.conf` faylida `SecRequestBodyAccess On` direktivasi yoqilgan bo‘lishi shart. Agar u `Off` bo‘lsa, ModSecurity faqat so‘rov sarlavhalarini ko‘radi va POST tanasidagi ma’lumotlarni filtrlay olmaydi.

7. **`1001302` istisnosi qaysi usul, URI, qoida va parametr bilan cheklangan?**
   - *Javob:*
     - Usul (Method): Faqat `POST`;
     - URI: Faqat `@streq /submit` (aniq `/submit` manzili);
     - Qoida: Faqat `1001301` IDli qoida;
     - Parametr: Faqat `ARGS:comment` maydoni (`ctl:ruleRemoveTargetById=1001301;ARGS:comment`).

8. **Nega T8ning 200 natijasidan keyin T9 va T10ni ham bajarish shart?**
   - *Javob:* T8 faqat qonuniy so‘rov o‘tganini ko‘rsatadi. Ammo ma’mur istisnoni haddan tashqari keng qilib, butun WAFni yoki boshqa sahifalarni himoyasiz qoldirgan bo‘lishi mumkin. T9 istisno boshqa endpointga (`/other`) tarqalmaganini, T10 esa aynan o‘sha `/submit` manzilida haqiqiy kiberhujumlarga qarshi CRS himoyasi buzilmaganini isbotlash uchun o‘ta muhimdir.

9. **HTTP 403 statusini aynan WAFga bog‘lash uchun statusdan tashqari qanday dalillar zarur?**
   - *Javob:* Faqat 403 statusining o‘zi yetarli emas, chunki 403 ni backend ilovaning o‘zi yoki Apache fayl huquqlari ham qaytarishi mumkin. WAF isbotlash uchun:
     1. Apache `a13-access.log` dagi o‘sha so‘rovning `UNIQUE_ID` kodi;
     2. Apache `a13-error.log` dagi ModSecurity qoida IDsi (`id "1001300"` yoki `id "949110"`);
     3. Apache `a13-audit.log` dagi to‘liq tahliliy yozuv;
     4. Backend jurnalida o‘sha so‘rovning umuman qayd etilmaganligi (so‘rov backendga yetmasdan WAFda to‘xtatilgani).

10. **WAFni butunlay o‘chirib false positive-ni "tuzatish" qanday halokatli oqibatlarga olib keladi?**
    - *Javob:* Bu tizimni "eshikni ochib qo‘yib, qulfni sindirib tashlash" bilan barobardir. Agar bitta foydalanuvchining so‘rovi o‘tishi uchun WAF o‘chirilsa (`SecRuleEngine Off`), butun veb-portal barcha turdagi SQL Injection, XSS, Remote Code Execution va Web Shell yuklash hujumlariga mutlaqo ochiq qoladi. To‘g‘ri muhandislik amaliyoti — bu faqatgina tor runtime istisnolar (tuning) orqali xavfsizlik va funksionallik muvozanatini saqlashdir.

---

## 5. FOYDALANILGAN RASMIY MANBALAR VA TAVSIYA ETILGAN ADABIYOTLAR

1. **Apache HTTP Server 2.4 — `mod_proxy` Rasmiy Hujjatlari:**
   - URL: [https://httpd.apache.org/docs/2.4/mod/mod_proxy.html](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html)
   - *Tavsif:* `ProxyRequests`, `ProxyPass`, `ProxyPassReverse`, `ProxyPreserveHost` direktivalari va Apache-da teskari proksi arxitekturasini loyihalash bo‘yicha to‘liq rasmiy qo‘llanma.

2. **OWASP Core Rule Set (CRS) — Extended Installation & Architecture:**
   - URL: [https://coreruleset.org/docs/1-getting-started/1-2-extended_install/](https://coreruleset.org/docs/1-getting-started/1-2-extended_install/)
   - *Tavsif:* WAF dvigatellari, CRS qoidalarini yuklash tartibi va xavfsizlik zanjirini barpo etish.

3. **OWASP ModSecurity v2.x — Rasmiy Reference Manual:**
   - URL: [https://github.com/owasp-modsecurity/ModSecurity/wiki/Reference-Manual-(v2.x)](https://github.com/owasp-modsecurity/ModSecurity/wiki/Reference-Manual-(v2.x))
   - *Tavsif:* `SecRuleEngine`, `SecRequestBodyAccess`, audit jurnallari turlari (`SecAuditLogParts`) va dinamik boshqaruv amallari (`ctl:ruleRemoveTargetById`, `chain`).

4. **OWASP CRS — False Positives and Tuning Guide:**
   - URL: [https://coreruleset.org/docs/2-how-crs-works/2-3-false-positives-and-tuning/](https://coreruleset.org/docs/2-how-crs-works/2-3-false-positives-and-tuning/)
   - *Tavsif:* Soxta ijobiy signallarni aniqlash, runtime istisnolar yozish va xavfsizlik qoidalarini nozik sozlash metodologiyasi.

5. **OWASP CRS — Anomaly Scoring System:**
   - URL: [https://coreruleset.org/docs/2-how-crs-works/2-1-anomaly_scoring/](https://coreruleset.org/docs/2-how-crs-works/2-1-anomaly_scoring/)
   - *Tavsif:* Kiruvchi va chiquvchi anomaliya ballarini yig‘ish mexanizmi, anomaliya chegaralari (Thresholds) va 949110 baholovchi qoidasining ishlash printsipi.

6. **IETF RFC 7230 / RFC 9110 — Hypertext Transfer Protocol (HTTP):**
   - URL: [https://www.rfc-editor.org/rfc/rfc9110.html](https://www.rfc-editor.org/rfc/rfc9110.html)
   - *Tavsif:* HTTP sarlavhalari, yo‘naltirishlar (Redirects, status 302), shlyuzlar (Gateways) va oraliq proksi vositalarining standart me’yorlari.
