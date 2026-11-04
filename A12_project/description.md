# TARMOQ XAVFSIZLIGI — A12 AMALIY MASHG‘ULOT
## DDoS HUJUMLARIDAN HIMOYALANISH CHORALARINI LOYIHALASH
### 1-VARIANT: Veb-xizmat uchun qatlamli himoya va nazoratli sinov

---

## 📌 KIRISH VA LABORATORIYA MUHITI HAQIDA MUHIM KO‘RSATMALAR

Ushbu qo‘llanma **Tarmoq xavfsizligi** fanining **A12 amaliy mashg‘uloti (1-variant)** bo‘yicha to‘liq va mukammal amaliy qo‘llanma hisoblanadi.

> [!IMPORTANT]
> **Talaba muhiti bo‘yicha muhim parametrlar va tuzatishlar:**
> 1. **Asosiy operatsion tizim (Host OS):** Windows emas, balki **Ubuntu Linux**.
> 2. **Virtualizatsiya tizimi (Hypervisor):** **VMware** (VMware Workstation Pro / Player for Linux).
> 3. **Virtual mashina (Guest VM):** **Ubuntu Server 24.04 LTS** (2 vCPU, 4 GB RAM, 25 GB disk).
> 4. **Tarmoq muhiti (Wi-Fi):** Universitet Wi-Fi tarmog‘idan emas, **shaxsiy lokal Wi-Fi** tarmog‘idan foydalaniladi.
>    * Dastlabki paketlarni (`nginx`, `python3`, `curl` va h.k.) o‘rnatish jarayonida VMware virtual mashinasi vaqtincha **NAT** adapterida bo‘ladi va internetdan paketlar yuklab olinadi.
>    * Paketlar o‘rnatilgach, VMning tashqi virtual tarmoq ulanishi (Cable connected) uziladi. Laboratoriya to‘liq **VM ichidagi izolyatsiyalangan virtual tarmoq (Linux Network Namespaces va Bridge)** orqali amalga oshiriladi. Shu sababli shaxsiy lokal Wi-Fi tarmog‘ingizga hech qanday xavf yoki ortiqcha yuklama tushmaydi.
> 5. **Bitta VM ichida ko‘p mijozli arxitektura:** Ushbu variantda bir nechta og‘ir virtual mashinalarni ishga tushirish o‘rniga, Linux yadrosining kuchli imkoniyati — **Network Namespace (`ip netns`)** texnologiyasi qo‘llaniladi. Bu bitta Ubuntu Server VM ichida bir-biridan to‘liq ajratilgan 2 ta alohida IP manzilga ega mijoz (`a12-good` va `a12-test`), Nginx reverse proxy va mahalliy backendni bir vaqtning o‘zida xavfsiz ishlatish imkonini beradi.

---

## 📑 MUNDARIJA
1. [1-QISM: Mavzuning Maqsadi va Tahdid Modeli](#1-qism-mavzuning-maqsadi-va-tahdid-modeli)
2. [2-QISM: Nazariy Tushunchalar va Texnologiyalar Qo‘llanmasi](#2-qism-nazariy-tushunchalar-va-texnologiyalar-qollanmasi)
3. [3-QISM: Mashqni Bajarish Bo‘yicha Bosqichma-bosqich Amaliy Qo‘llanma](#3-qism-mashqni-bajarish-boyicha-bosqichma-bosqich-amaliy-qollanma)
4. [4-QISM: Shartli Ishlab Chiqarish Ssenariysi va Hisob-kitoblar (6.1-bo‘lim yechimlari)](#4-qism-shartli-ishlab-chiqarish-ssenariysi-va-hisob-kitoblar-61-bolim-yechimlari)
5. [5-QISM: Veb-portal Uchun DDoS Himoya Loyihasi va Hodisaga Javob Rejasi (Incident Response)](#5-qism-veb-portal-uchun-ddos-himoya-loyihasi-va-hodisaga-javob-rejasi-incident-response)
6. [6-QISM: O‘qituvchi Uchun Himoya Savollari va Mukammal Javoblar (12 ta savol)](#6-qism-oqituvchi-uchun-himoya-savollari-va-mukammal-javoblar-12-ta-savol)
7. [7-QISM: Nosozliklarni Bartaraf Etish (Troubleshooting) va Labni Tozalash](#7-qism-nosozliklarni-bartaraf-etish-troubleshooting-va-labni-tozalash)
8. [8-QISM: Foydalanilgan Rasmiy va Ilmiy Manbalar](#8-qism-foydalanilgan-rasmiy-va-ilmiy-manbalar)

---

# 1-QISM: MAVZUNING MAQSADI VA TAHDID MODELI

### 1.1. Mavzuning Asosiy Maqsadi
Axborot xavfsizligining klassik **CIA triadasida** (Confidentiality, Integrity, **Availability** — Maxfiylik, Butunlik, **Mavjudlik**) eng ko‘p hujumga uchraydigan jihatlardan biri bu **Mavjudlik (Availability)** hisoblanadi.

Ushbu laboratoriya ishining maqsadi:
* Veb-xizmatlarning mavjudligiga bo‘ladigan tahdidlarni to‘g‘ri tasniflash;
* **Nginx reverse proxy** yordamida L7 (Application layer) darajasida HTTP so‘rovlari tezligini cheklash (**Rate Limiting**);
* Himoyani dastlab passiv kuzatuv (**dry_run**) rejimida sinab, so‘ngra faol bloklash rejimiga o‘tkazish;
* Qonuniy oddiy foydalanuvchilar (`a12-good`) hamda tajovuzkor/tezkor yuklama beruvchi mijozlar (`a12-test`) oqimini bir vaqtning o‘zida o‘tkazib, himoyaning samaradorligini dalillar (CSV va JSON loglar) asosida tahlil qilish;
* Amaliy tajriba cheklovlarini anglagan holda, haqiqiy ishlab chiqarish (production) muhiti uchun **qatlamli himoya (Defense-in-Depth)** va **hodisaga javob berish (Incident Response)** rejasini ishlab chiqish.

### 1.2. DoS va DDoS: Fundamental Farq
* **DoS (Denial of Service — Xizmatni rad etish):** Bitta manbadan nishon serverga qaratilgan hujum. Maqsad — server resurslarini (CPU, xotira, tarmoq kanali, ulanishlar jadvali) to‘ldirib, qonuniy foydalanuvchilarga xizmat ko‘rsatishni to‘xtatish.
* **DDoS (Distributed Denial of Service — Taqsimlangan xizmatni rad etish):** Dunyo bo‘ylab tarqalgan yuzlab yoki millionlab qurilmalar (botnetlar, kompromiss qilingan IoT qurilmalar) orqali bitta nishonga bir vaqtda hujum uyushtirish.
* **Ushbu laboratoriyaning chegarasi:** Bu laboratoriya nazoratli va lokal sinov bo‘lib, unda to‘liq hajmli DDoS o‘tkazilmaydi. Laboratoriyada bitta tezkor manba va bitta oddiy manba ishtirok etadi. Biroq, ushbu tajriba orqali har bir alohida IP yoki global limit qanday ishlashi va uning qayerda ojiz qolishi matematik hamda amaliy jihatdan isbotlanadi.

### 1.3. Tahdid Modeli va Cheklangan Resurslar Qatlamlari
DDoS hujumlari OSI modelining turli qatlamlariga qaratiladi:

| Qatlam / Tahdid turi | Hujum usuli misollari | Asosiy cheklangan resurs | Himoya joylashuvi |
| :--- | :--- | :--- | :--- |
| **Hajmli L3/L4 trafik (Volumetric)** | UDP Amplification, NTP/DNS reflection, ICMP flood | Internet kanalining bit/s yoki paket/s sig‘imi (Bandwidth) | Internet provayder (ISP), Magistral tarmoq, Trafikni tozalash markazlari (Scrubbing Centers, Cloudflare, Akamai) |
| **Ulanish holatiga bosim (State Exhaustion)** | SYN Flood, TCP FIN/RST flood, SSL/TLS handshake flood | TCP holat jadvali (conntrack), server xotirasi, xavfsizlik devori (Firewall) ulanishlar sig‘imi | Chegara qurilmalari, apparat xavfsizlik devori, Linux yadrosi sozlamalari (`syncookies`) |
| **Ko‘p HTTP so‘rovi (L7 Application)** | HTTP GET/POST Flood, Slowloris, murakkab DB so‘rovlarini chaqirish | Ilova ishchilari (workers), Web Server CPU/RAM, Ma’lumotlar bazasi (DB connection pool) | Reverse Proxy (Nginx, HAProxy), WAF (Web Application Firewall), Ilova ichki siyosati |

> [!WARNING]
> Katta trafik paydo bo‘lishi har doim ham DDoS hujumini anglatmaydi! Universitetda imtihon natijalari e’lon qilinganda yoki qabul boshlanganda minglab talabalar bir vaqtda portalga kirishi mumkin (**Flash Crowd** / Qonuniy talabning keskin oshishi). Shu sababli, xavfsizlik choralari qonuniy foydalanuvchilarni bloklab qo‘ymasligi (False Positive) shart.

---

# 2-QISM: NAZARIY TUSHUNCHALAR VA TEXNOLOGIYALAR QO‘LLANMASI

### 2.1. Linux Network Namespaces (`netns`), `veth` va Linux Bridge
Laboratoriyada bir nechta jismoniy kompyuter yoki og‘ir virtual mashinalar yaratish o‘rniga Linux yadrosining virtualizatsiya imkoniyatlaridan foydalaniladi:
1. **Network Namespace (`netns`):** Linux operatsion tizimida mustaqil tarmoq steki (o‘zining IP manzillari, marshrutlash (routing) jadvali, ARP jadvali va soketlari) ga ega bo‘lgan ajratilgan muhit. Biz ikkita namespace yaratamiz:
   * `a12-good`: 10.120.10.11 IP manziliga ega oddiy talabani ifodalaydi.
   * `a12-test`: 10.120.10.12 IP manziliga ega tezkor sinov yuklamasini (tajovuzkor/bot) ifodalaydi.
2. **Virtual Ethernet (`veth`) juftligi:** Virtual tarmoq kabeli. Uning bir uchi asosiy tizimga (yoki bridgega), ikkinchi uchi esa virtual namespace ichiga ulanadi. Bir uchidan kirgan paket ikkinchi uchidan chiqadi (`v-good` <-> `c-good`).
3. **Linux Bridge (`br-a12`):** Virtual tarmoq kommutatori (switch). U 10.120.10.1 IP manzilini oladi va Nginx aynan shu interfeysda mijozlar so‘rovini qabul qiladi.

```
+---------------------------------------------------------------------------------+
|                               Ubuntu Server VM                                  |
|                                                                                 |
|   +-----------------------+                         +-----------------------+   |
|   |   netns: a12-good     |                         |   netns: a12-test     |   |
|   |     10.120.10.11      |                         |     10.120.10.12      |   |
|   |       [ c-good ]      |                         |       [ c-test ]      |   |
|   +-----------|-----------+                         +-----------|-----------+   |
|               | (veth)                                          | (veth)        |
|               |                                                 |               |
|   +-----------|-------------------------------------------------|-----------+   |
|   |       [ v-good ]                                        [ v-test ]      |   |
|   |                                                                         |   |
|   |                        Linux Bridge: br-a12                             |   |
|   |                           10.120.10.1/24                                |   |
|   +---------------------------------|---------------------------------------+   |
|                                     |                                           |
|                                     v                                           |
|                           +-------------------+                                 |
|                           |   Nginx Proxy     |                                 |
|                           |  10.120.10.1:80   |                                 |
|                           +---------|---------+                                 |
|                                     | proxy_pass (Loopback)                     |
|                                     v                                           |
|                           +-------------------+                                 |
|                           |  Python Backend   |                                 |
|                           |  127.0.0.1:8080   |                                 |
|                           +-------------------+                                 |
+---------------------------------------------------------------------------------+
```

### 2.2. Nginx Reverse Proxy va Backend Izolyatsiyasi
* **Reverse Proxy:** Foydalanuvchilar va haqiqiy backend (ilova) o‘rtasida turuvchi oraliq server. Mijozlar bevosita backend bilan emas, Nginx bilan muloqot qiladi.
* **Origin izolyatsiyasi:** Backend faqat `127.0.0.1:8080` (loopback) manzilida tinglaydi. U tashqi tarmoqqa (`0.0.0.0` yoki `10.120.10.1`) ochilmaydi. Agar tajovuzkor Nginx himoyasini aylanib o‘tish (bypass) uchun to‘g‘ridan-to‘g‘ri `10.120.10.1:8080` ga murojaat qilsa, tarmoq darajasida ulanish rad etiladi.

### 2.3. Nginx Rate Limiting algoritmi: Leaky Bucket (Oqayotgan chelak)
Nginx so‘rovlar tezligini cheklash uchun mashhur **Leaky Bucket (Oqayotgan chelak)** algoritmidan foydalanadi.
* Tasavvur qiling, chelak tubida kichik teshik bor va suv (so‘rovlar) doimiy belgilangan tezlikda oqib chiqadi (`rate=2r/s` — har 0.5 soniyada 1 ta so‘rov).
* Agar suv chelakka belgilangan tezlikdan tezroq quyilsa, ortiqcha suv chelak hajmiga (`burst=4`) qadar to‘planadi.
* Agar chelak to‘lib toshsa, yangi kelgan suv darhol to‘kiladi (so‘rov darhol rad etiladi — `429 Too Many Requests`).
* **`nodelay` parametri:** Odatda `burst` ichiga tushgan so‘rovlar navbatda kutib turadi va sekin o‘tkaziladi. Ammo `nodelay` kaliti qo‘shilsa, chelak hajmidagi (burst) so‘rovlar sun’iy kechiktirilmasdan zudlik bilan backendga yuboriladi, faqat chelak to‘lgandan keyingi so‘rovlargina darhol 429 xatosini oladi.

### 2.4. Nginx Konfiguratsiya Direktivalari
* `limit_req_zone $binary_remote_addr zone=a12_ip:1m rate=2r/s;`:
  - `$binary_remote_addr`: Mijozning IP manzili xotirada joy tejash uchun binar shaklda saqlanadi (IPv4 uchun atigi 4 bayt, oddiy `$remote_addr` esa 7-15 bayt egallaydi).
  - `zone=a12_ip:1m`: Mijozlar holatini saqlash uchun `a12_ip` nomli 1 Megabaytli umumiy xotira (Shared Memory) ajratiladi. 1 MB xotira taxminan 16,000 ta turli IP manzil holatini saqlay oladi.
  - `rate=2r/s`: Har bir alohida IP uchun ruxsat etilgan tezlik — sekundiga 2 ta so‘rov.
* `limit_req zone=a12_ip burst=4 nodelay;`: Belgilangan cheklovni joriy `location /` blokiga tatbiq etish.
* `limit_req_status 429;`: Standart 503 (Service Temporarily Unavailable) o‘rniga RFC 6585 bo‘yicha to‘g‘ri HTTP status kodi — **429 (Too Many Requests)** qaytarish. Bu orqali xizmat nosozligi (5xx) bilan xavfsizlik cheklovi (429) bir-biridan aniq ajraladi.
* `limit_req_dry_run on;`: Kuzatuv rejimi. Nginx so‘rovni limitdan oshganini aniqlaydi, logga `REJECTED_DRY_RUN` deb yozadi, ammo so‘rovni amalda **bloklamaydi**, balki backendga o‘tkazib yuboradi (status 200). Bu rejim yangi siyosatni ishlab chiqarishga joriy etishdan oldin uning ta’sirini baholash uchun zarur.
* `limit_req_dry_run off;`: Faol cheklov rejimi. Limitdan oshgan so‘rovlar darhol bloklanadi va backendga yetib bormasdan Nginxning o‘zidan 429 kodi qaytariladi.

### 2.5. Log Tahlili va JSON Format
Nginx so‘rovlarni tahlil qilish uchun strukturalangan JSON formatidan foydalanadi:
* `ip`: So‘rov yuborgan mijozning IP manzili (`$remote_addr`).
* `status`: Mijozga qaytgan HTTP javob kodi (masalan, 200 yoki 429).
* `rt`: So‘rovni bajarishning to‘liq vaqti soniyalarda (`$request_time`).
* `up`: Backend (upstream) qaytargan holat kodi (`$upstream_status`). Agar so‘rov Nginx tomonidan rad etilsa, backendga umuman bormaydi va `up` qiymati `-` (tire) bo‘ladi.
* `limit`: Rate limiting modulining qarori (`$limit_req_status`). Mumkin bo‘lgan qiymatlar:
  - `PASSED`: So‘rov limitdan oshmadi, ruxsat berildi.
  - `DELAYED`: Burst orqali kechiktirildi (nodelay bo‘lmaganda).
  - `REJECTED_DRY_RUN`: Kuzatuv rejimida rad etilishi kerak edi, ammo o‘tkazib yuborildi.
  - `REJECTED`: Faol rejimda so‘rov amalda rad etildi (429 qaytarildi).

### 2.6. Mijoz IP Manziliga Ishonch va X-Forwarded-For Xavfi
Agar server CDN (masalan, Cloudflare) yoki yuklama taqsimlagich (Load Balancer) ortida tursa, Nginx ko‘radigan `$remote_addr` haqiqiy mijozning IPsi emas, balki proksi serverning IPsi bo‘lib qoladi.
Haqiqiy mijoz IPsi `X-Forwarded-For` HTTP sarlavhasida (header) uzatiladi.
**Biroq ogohlantirish:** Agar siz shunchaki ixtiyoriy mijoz yuborgan `X-Forwarded-For` sarlavhasiga ishonsangiz, tajovuzkor o‘z so‘rovlariga soxta IP yozib (`X-Forwarded-For: 1.2.3.4`), har bir so‘rovda IPni o‘zgartirib rate limitni osongina aylanib o‘tishi mumkin (**IP Spoofing**).
Shuning uchun Nginxda `ngx_http_realip_module` orqali faqat **aniq ishonchli proksilar ro‘yxati (`set_real_ip_from`)** belgilanadi va begona mijozlar sarlavhalari inobatga olinmaydi!

---

# 3-QISM: MASHQNI BAJARISH BO‘YICHA BOSQICHMA-BOSQICH AMALIY QO‘LLANMA

Ushbu qismda har bir buyruq, konfiguratsiya fayli va kodlar ketma-ketlikda bajariladi. Har bir buyruqning nima vazifa bajarishi qat’iy izohlab berilgan.

### 3.1. 0-bosqich: Dastlabki Tayyorgarlik va Paketlarni O‘rnatish

Ubuntu tizimida terminalni oching. VMware VM sozlamalarida tarmoq kartasi (Network Adapter) **NAT** rejimida ekanligini tekshiring (paketlarni internetdan yuklash uchun).

VM konsolida quyidagi buyruqlarni bajaring:

```bash
sudo apt update
sudo apt install -y nginx python3 iproute2 curl nano
sudo systemctl stop nginx
nginx -v
python3 --version
```

🔍 **Har bir buyruqning vazifasi:**
* `sudo apt update`: Ubuntu repozitoriylaridagi paketlar ro‘yxatini va so‘nggi versiyalar ma’lumotlarini yangilaydi.
* `sudo apt install -y nginx python3 iproute2 curl nano`: Laboratoriya uchun kerakli vositalarni o‘rnatadi:
  - `nginx`: Veb-server va reverse proxy;
  - `python3`: Backend server va sinov probasi (`probe.py`) uchun dasturlash tili;
  - `iproute2`: Virtual tarmoq, bridge va namespace boshqaruv buyruqlari (`ip` utilitasi);
  - `curl`: HTTP so‘rovlarini terminaldan yuborish va test qilish vositasi;
  - `nano`: Matn muharriri.
* `sudo systemctl stop nginx`: Tizim bilan birga avtomatik ishga tushgan asosiy Nginx xizmatini to‘xtatadi. Chunki biz tizimning global konfiguratsiyasiga tegmasdan, `/opt/a12/nginx.conf` da alohida mustaqil Nginx nusxasini ishlatamiz.
* `nginx -v`: Nginx o‘rnatilganini va uning versiyasini tekshiradi.
* `python3 --version`: Python o‘rnatilganini tekshiradi (Python 3.10+ bo‘lishi lozim).

> [!IMPORTANT]
> **Kabelni uzish:** Paketlar o‘rnatib bo‘lingach, VMware menyusidan VM sozlamalariga kiring va Network Adapter dagi **"Connected"** hamda **"Connect at power on"** belgilarini olib tashlang (tarmoq kabelini uzing). Konsol ishlashda davom etadi. Bu tashqi tarmoq aralashuvining oldini oladi.
> 
> **Snapshot olish:** VMware menyusidan **VM -> Snapshot -> Take Snapshot** tugmasini bosing va nomini `A12-base` deb belgilang. Agar xatolikka yo‘l qo‘ysangiz, bir zumda toza holatga qaytishingiz mumkin bo‘ladi.

---

### 3.2. 1-bosqich: Virtual Tarmoq va Alohida Mijozlarni Yaratish

Root foydalanuvchisi rejimiga o‘tamiz va virtual tarmoq infratuzilmasini quramiz:

```bash
sudo -i
mkdir -p /opt/a12/results
ip link add br-a12 type bridge
ip addr add 10.120.10.1/24 dev br-a12
ip link set br-a12 up
```

🔍 **Buyruqlar tahlili:**
* `sudo -i`: Root (administrator) sessiyasini ochadi. Keyingi barcha amallar shu sessiyada bajariladi.
* `mkdir -p /opt/a12/results`: Laboratoriya fayllari, konfiguratsiyalar va natijaviy CSV fayllar saqlanadigan `/opt/a12/results` katalogini yaratadi.
* `ip link add br-a12 type bridge`: Tizimda `br-a12` nomli yangi virtual tarmoq ko‘prigi (bridge) yaratadi.
* `ip addr add 10.120.10.1/24 dev br-a12`: Ushbu bridge interfeysiga `10.120.10.1` IP manzilini va 24-bitli subnet maskani (`255.255.255.0`) biriktiradi.
* `ip link set br-a12 up`: `br-a12` interfeysini faol (UP) holatga keltiradi.

Endi ikkita mijoz uchun namespace va virtual kabel (veth) larni sikl orqali yaratamiz:

```bash
for item in good:11 test:12; do
  name=${item%:*}
  octet=${item#*:}
  ip netns add a12-$name
  ip link add v-$name type veth peer name c-$name
  ip link set v-$name master br-a12
  ip link set v-$name up
  ip link set c-$name netns a12-$name
  ip -n a12-$name link set lo up
  ip -n a12-$name link set c-$name up
  ip -n a12-$name addr add 10.120.10.$octet/24 dev c-$name
done
```

🔍 **Sikl ichidagi har bir buyruqning vazifasi:**
* `for item in good:11 test:12; do ... done`: Sikl ikki marta aylanadi: birinchi `good` (IP oxiri 11), keyin `test` (IP oxiri 12).
* `ip netns add a12-$name`: Yangi izolyatsiyalangan tarmoq namespace'ini yaratadi (`a12-good` va `a12-test`).
* `ip link add v-$name type veth peer name c-$name`: Virtual Ethernet juftligini yaratadi. Masalan: `v-good` (asosiy tizim tomoni) va `c-good` (mijoz tomoni).
* `ip link set v-$name master br-a12`: Virtual kabelning server tomonini (`v-good`) `br-a12` ko‘prigiga (bridge portiga) ulaydi.
* `ip link set v-$name up`: Kabelning server uchini yoqadi.
* `ip link set c-$name netns a12-$name`: Virtual kabelning mijoz tomonini (`c-good`) tegishli namespace (`a12-good`) ichiga ko‘chiradi.
* `ip -n a12-$name link set lo up`: Namespace ichidagi mahalliy loopback (`lo`) interfeysini yoqadi.
* `ip -n a12-$name link set c-$name up`: Namespace ichidagi mijoz interfeysini (`c-good`) faollashtiradi.
* `ip -n a12-$name addr add 10.120.10.$octet/24 dev c-$name`: Mijoz interfeysiga IP manzil biriktiradi (`10.120.10.11/24` va `10.120.10.12/24`).

**Yaratilgan tarmoqni tekshiramiz:**

```bash
ip -br addr show br-a12
ip -n a12-good route
ip -n a12-test route
```

🔍 **Natijani tahlil qilish:**
* `br-a12` interfeysi `UP` va `10.120.10.1/24` manziliga ega bo‘lishi kerak.
* Har ikkala namespace ichida faqat `10.120.10.0/24 dev c-...` lokal marshruti chiqishi kerak.
* **Muhim:** Hech qanday `default via ...` (default gateway) yozuvi bo‘lmasligi kerak! Mijozlar faqat bridge ichidagi 10.120.10.0/24 tarmog‘iga bog‘langan bo‘lib, tashqi dunyoga chiqa olmaydi.

---

### 3.3. 2-bosqich: Test Backend Ilovasini Yaratish va Ishga Tushirish

Haqiqiy veb-xizmat vazifasini bajaruvchi kichik Python HTTP serverini yaratamiz. Har bir GET so‘rovi uchun u 0.2 soniya kutadi (`time.sleep(0.2)`) va javob qaytaradi. Bu xizmatning ishlash vaqtini va yuklama ostidagi javob kechikishini modellashtiradi.

Faylni yaratamiz:

```bash
cat > /opt/a12/backend.py <<'APP'
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
import time

class App(BaseHTTPRequestHandler):
    def do_GET(self):
        time.sleep(0.2)
        body = b"A12 backend OK\n"
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

ThreadingHTTPServer(("127.0.0.1", 8080), App).serve_forever()
APP
chmod 644 /opt/a12/backend.py
```

🔍 **Backend kodi tahlili:**
* `ThreadingHTTPServer(("127.0.0.1", 8080), App)`: Server faqat `127.0.0.1` (localhost) va `8080` portida tinglaydi. Har bir kelgan so‘rov alohida oqimda (thread) ko‘rib chiqiladi.
* `time.sleep(0.2)`: Har bir so‘rovga 200 millisekundlik sun’iy ishlov berish kechikishini kiritadi.
* `self.send_response(200)`: HTTP 200 OK maqom kodini yuboradi.
* `chmod 644`: Faylga barcha foydalanuvchilar o‘qiy oladigan, lekin faqat root o‘zgartira oladigan huquq beradi.

Ushbu backendni Linux `systemd` xizmati (daemon) sifatida doimiy fonda ishlaydigan qilamiz:

```bash
cat > /etc/systemd/system/a12-backend.service <<'UNIT'
[Unit]
Description=A12 local teaching backend

[Service]
User=www-data
ExecStart=/usr/bin/python3 /opt/a12/backend.py
Restart=on-failure
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
UNIT
```

🔍 **Systemd servisi direktivalari tahlili:**
* `User=www-data`: Xavfsizlik qoidasi — backend xizmati imtiyozli root foydalanuvchisi sifatida emas, cheklangan `www-data` foydalanuvchisi nomidan ishga tushadi.
* `ExecStart=/usr/bin/python3 /opt/a12/backend.py`: Xizmat ishga tushganda bajariladigan aniq buyruq.
* `Restart=on-failure`: Agar xizmat biron sabab bilan to‘xtab qolsa, systemd uni avtomatik qayta ishga tushiradi.
* `NoNewPrivileges=true` va `PrivateTmp=true`: Jarayonning qo‘shimcha tizim imtiyozlarini olishini bloklaydi va alohida izolyatsiyalangan `/tmp` katalogini beradi.

Servisni faollashtiramiz va tekshiramiz:

```bash
systemctl daemon-reload
systemctl start a12-backend
systemctl is-active a12-backend
curl --noproxy '*' --max-time 3 http://127.0.0.1:8080/
ss -lntp 'sport = :8080'
```

🔍 **Tekshirish natijalari:**
* `systemctl is-active a12-backend` buyrug‘i `active` javobini qaytarishi kerak.
* `curl` buyrug‘i `A12 backend OK` javobini qaytarishi kerak.
* `ss -lntp 'sport = :8080'` buyrug‘i natijasida port tinglovchisi **faqat** `127.0.0.1:8080` bo‘lishi shart! Agar u yerda `0.0.0.0:8080` yoki `10.120.10.1:8080` chiqsa, bu xato hisoblanadi (chunki tashqi foydalanuvchilar Nginxni aylanib o‘tishi mumkin bo‘lib qoladi).

---

### 3.4. 3-bosqich: Nginx Konfiguratsiyasini Yaratish (Kuzatuv Rejimi)

Biz tizimning `/etc/nginx/nginx.conf` fayliga tegmaymiz. Laboratoriya uchun `/opt/a12/nginx.conf` da alohida to‘liq konfiguratsiya yaratamiz.

Konfiguratsiya faylini yozamiz:

```bash
cat > /opt/a12/nginx.conf <<'CONF'
user www-data;
worker_processes 1;
pid /run/a12-nginx.pid;
error_log /opt/a12/error.log notice;

events { 
    worker_connections 256; 
}

http {
    log_format a12 escape=json
      '{"time":"$time_iso8601","ip":"$remote_addr",'
      '"uri":"$request_uri","status":$status,'
      '"rt":$request_time,"up":"$upstream_status",'
      '"limit":"$limit_req_status"}';

    access_log /opt/a12/access.jsonl a12;

    limit_req_zone $binary_remote_addr zone=a12_ip:1m rate=2r/s;

    server {
        listen 10.120.10.1:80;
        server_name a12.test;

        client_max_body_size 16k;
        client_header_timeout 5s;
        client_body_timeout 5s;
        keepalive_timeout 5s;

        location / {
            limit_req zone=a12_ip burst=4 nodelay;
            limit_req_status 429;
            limit_req_log_level notice;
            limit_req_dry_run on;

            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;

            proxy_connect_timeout 1s;
            proxy_read_timeout 3s;
            proxy_send_timeout 3s;
        }
    }
}
CONF
```

🔍 **Nginx konfiguratsiyasi direktivalari tahlili:**
* `user www-data; worker_processes 1;`: Nginx ishchi jarayoni `www-data` nomidan ishlaydi, 1 ta yadro ajratiladi.
* `pid /run/a12-nginx.pid;`: Jarayon ID fayli.
* `log_format a12 escape=json ...`: Loglarni JSON qatorlarida (`.jsonl`) saqlash formati. Har bir so‘rov bo‘yicha vaqt, mijoz IPsi, so‘rov manzili, HTTP maqom kodi (`status`), bajarilish vaqti (`rt`), backend holati (`up`) va limit holati (`limit`) yoziladi.
* `limit_req_zone $binary_remote_addr zone=a12_ip:1m rate=2r/s;`: Har bir alohida mijoz IPsi uchun 2 ruxsat/sekund tezlikda 1 MB umumiy xotiraga ega `a12_ip` nomli cheklov zonasi.
* `listen 10.120.10.1:80;`: Nginx faqat virtual bridge IPsi va 80-portda so‘rov kutadi.
* `limit_req zone=a12_ip burst=4 nodelay;`: 4 ta so‘rovlik portlash (burst) ruxsati beriladi va `nodelay` orqali kechiktirilmasdan bajariladi.
* `limit_req_status 429;`: Limitdan oshgan so‘rovlarga 429 xatosi qaytariladi.
* `limit_req_dry_run on;`: **Dastlabki kuzatuv rejimi!** Qoidani buzgan so‘rovlar amalda bloklanmaydi, faqat logga qayd etiladi.
* `proxy_pass http://127.0.0.1:8080;`: So‘rovlarni mahalliy backendga yo‘naltiradi.
* `proxy_connect_timeout 1s; proxy_read_timeout 3s;`: Backend bilan ulanish va undan javob o‘qish timeoutlari.

Nginx konfiguratsiyasini tekshiramiz va ishga tushiramiz:

```bash
nginx -t -c /opt/a12/nginx.conf
nginx -c /opt/a12/nginx.conf
ip netns exec a12-good curl --noproxy '*' --max-time 3 -i http://10.120.10.1/?run=setup
ss -lntp 'sport = :80'
```

🔍 **Natijani tekshirish:**
* `nginx -t ...`: `syntax is ok` va `test is successful` xabarini chiqarishi kerak.
* `nginx -c /opt/a12/nginx.conf`: Nginxni aynan shu maxsus konfiguratsiya bilan ishga tushiradi.
* `ip netns exec a12-good curl ...`: `a12-good` namespace'i ichidan Nginxga birinchi test so‘rovini yuboradi. Natijada `HTTP/1.1 200 OK` va `A12 backend OK` chiqishi kerak.
* `ss -lntp 'sport = :80'`: 80-portda Nginx `10.120.10.1:80` manzilida tinglayotganini ko‘rsatishi kerak.

---

### 3.5. 4-bosqich: Sinov Probasi Dasturini (`probe.py`) Yaratish

Nazoratli va aniq o‘lchovli test o‘tkazish uchun Python dasturini tayyorlaymiz.

```bash
cat > /opt/a12/probe.py <<'PROBE'
import csv, sys, time
from concurrent.futures import ThreadPoolExecutor
from urllib.request import build_opener, ProxyHandler, Request
from urllib.error import HTTPError

mode, tag, out = sys.argv[1:]
if mode not in ("normal", "fast") or tag not in ("before", "after"):
    raise SystemExit("Rejim yoki bosqich noto'g'ri")

url = "http://10.120.10.1/?run=" + tag

def one(i):
    start = time.monotonic()
    status = "ERR"
    try:
        op = build_opener(ProxyHandler({}))
        req = Request(url, headers={"Connection": "close"})
        with op.open(req, timeout=3) as response:
            status = str(response.status)
            response.read()
    except HTTPError as exc:
        status = str(exc.code)
        exc.read()
        exc.close()
    except Exception as exc:
        status = "ERR_" + type(exc).__name__
    ms = round((time.monotonic() - start) * 1000, 2)
    return [i, status, ms]

begin = time.monotonic()
if mode == "fast":
    with ThreadPoolExecutor(max_workers=4) as pool:
        rows = list(pool.map(one, range(1, 41)))
else:
    rows = []
    for i in range(1, 9):
        rows.append(one(i))
        if i < 8:
            time.sleep(1)

with open(out, "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["id", "status", "ms"])
    writer.writerows(rows)

print(mode, tag, "N=", len(rows), "elapsed_s=", round(time.monotonic() - begin, 2))
PROBE
```

🔍 **Dastur kodi tahlili:**
* Dastur 3 ta argument qabul qiladi: `mode` (`normal` yoki `fast`), `tag` (`before` yoki `after`), `out` (natija yoziladigan CSV fayl yo‘li).
* `normal` rejim: 8 ta ketma-ket so‘rov yuboradi. Har bir so‘rov orasida 1 soniya kutadi (`time.sleep(1)`). Bu qonuniy oddiy talabani ifodalaydi.
* `fast` rejim: 4 ta parallel ishchi oqim (`ThreadPoolExecutor(max_workers=4)`) bilan bir vaqtda jami 40 ta so‘rov yuboradi. Bu limitdan oshib ketuvchi tezkor yuklamani (tajovuzkorni) ifodalaydi.
* `one(i)` funksiyasi: Har bir so‘rovning boshlanish va tugash vaqtini aniq o‘lchaydi (`time.monotonic()`), HTTP javob kodini aniqlaydi va natijani `[id, status, ms]` ko‘rinishida CSV faylga saqlaydi.

Kodni sintaktik xatolarga tekshirib olamiz:

```bash
python3 -m py_compile /opt/a12/backend.py /opt/a12/probe.py
```
*(Agar xatolik chiqmasa, fayllar to‘g‘ri yozilgan).*

---

### 3.6. 5-bosqich: "Oldin" (Before) Bosqichi — Kuzatuv Rejimida Sinov

Bu bosqichda Nginx `dry_run on` rejimida turibdi. Biz bir vaqtning o‘zida ikkala mijozdan so‘rov yuboramiz.

```bash
sleep 5
timeout 45s ip netns exec a12-good python3 /opt/a12/probe.py normal before /opt/a12/results/good-before.csv &
GOOD_PID=$!
timeout 45s ip netns exec a12-test python3 /opt/a12/probe.py fast before /opt/a12/results/test-before.csv
wait "$GOOD_PID"
grep 'REJECTED' /opt/a12/access.jsonl | tail -n 6
```

🔍 **Buyruqlar tahlili:**
* `sleep 5`: Avvalgi sinovlardan qolgan hisoblagichlar tozalanishi uchun 5 soniya kutadi.
* `ip netns exec a12-good ... &`: `a12-good` namespace'ida oddiy mijozni fonda (`&`) ishga tushiradi. U 8 soniya davomida 8 ta so‘rov yuboradi.
* `GOOD_PID=$!`: Fonda ishlayotgan jarayonning ID raqamini saqlab oladi.
* `ip netns exec a12-test ...`: Shu vaqtning o‘zida `a12-test` namespace'ida 40 ta parallel so‘rov yuboruvchi tezkor mijozni ishga tushiradi.
* `wait "$GOOD_PID"`: Har ikkala mijoz o‘z ishini to‘liq tugatishini kutadi.
* `grep 'REJECTED' /opt/a12/access.jsonl | tail -n 6`: Log fayldan rad etish holatlarini qidiradi.

📊 **Kutiladigan natija:**
* `good-before.csv`: 8 ta so‘rovning barchasi `status: 200` bo‘ladi.
* `test-before.csv`: 40 ta so‘rovning barchasi `status: 200` bo‘ladi (chunki dry_run yoqilgan).
* `access.jsonl` loglarida: Tezkor mijoz so‘rovlari uchun `"status":200`, `"up":"200"`, ammo `"limit":"REJECTED_DRY_RUN"` yozuvi chiqadi. Bu shuni anglatadiki: Nginx agar cheklov yoqilganda ushbu so‘rovlarni bloklagan bo‘lar edi, lekin kuzatuv rejimida bo‘lgani uchun ularni backendga o‘tkazdi va mijoz 200 oldi.

---

### 3.7. 6-bosqich: "Keyin" (After) Bosqichi — Faol Cheklovni Yoqish

Endi kuzatuv rejimini o‘chirib, faol cheklovni (`dry_run off`) yoqamiz va xuddi shu sinovni takrorlaymiz.

```bash
cp /opt/a12/nginx.conf /opt/a12/nginx-before.conf
sed -i 's/limit_req_dry_run on;/limit_req_dry_run off;/' /opt/a12/nginx.conf
nginx -t -c /opt/a12/nginx.conf && nginx -s reload -c /opt/a12/nginx.conf
```

🔍 **Buyruqlar tahlili:**
* `cp ...`: Eski konfiguratsiyaning zaxira nusxasini saqlaydi.
* `sed -i 's/limit_req_dry_run on;/limit_req_dry_run off;/' ...`: Fayl ichidagi `limit_req_dry_run on;` qatorini `limit_req_dry_run off;` ga almashtiradi.
* `nginx -t ... && nginx -s reload ...`: Sintaksis to‘g‘ri bo‘lsa, Nginxni to‘xtatmasdan yangi konfiguratsiyani yuklaydi (Hot Reload).

Endi sinovni takrorlaymiz:

```bash
sleep 5
timeout 45s ip netns exec a12-good python3 /opt/a12/probe.py normal after /opt/a12/results/good-after.csv &
GOOD_PID=$!
timeout 45s ip netns exec a12-test python3 /opt/a12/probe.py fast after /opt/a12/results/test-after.csv
wait "$GOOD_PID"
grep 'REJECTED' /opt/a12/access.jsonl | tail -n 8
```

📊 **Kutiladigan natija:**
* `good-after.csv`: Oddiy mijoz har soniyada 1 tadan so‘rov yuborgani uchun u `rate=2r/s` limitidan oshmaydi va uning **barcha 8 ta so‘rovi yana 200 OK** oladi!
* `test-after.csv`: Tezkor mijoz dastlabki burst (4 ta) va joriy sekunddagi kvotadan (2 ta) o‘tgach, qolgan so‘rovlariga **429 (Too Many Requests)** javobini oladi!
* `access.jsonl` loglarida: 429 olgan qatorlarda `"status":429`, `"up":"-"`, `"limit":"REJECTED"` bo‘ladi. `"up":"-"` belgisi so‘rov backendga umuman yetib bormagani va backend resurslari tejalganini isbotlaydi!

---

### 3.8. 7-bosqich: Natijalarni Umumlashtirish va Hisoblash

Natijalarni hisoblash uchun quyidagi skriptni bajaramiz:

```bash
python3 - <<'METRICS'
import csv, glob, math
from collections import Counter

for file in sorted(glob.glob('/opt/a12/results/*.csv')):
    with open(file) as f:
        rows = list(csv.DictReader(f))
    n = len(rows)
    counts = Counter(r['status'] for r in rows)
    ok = sorted(float(r['ms']) for r in rows if r['status'] == '200')
    p95 = ok[math.ceil(0.95 * len(ok)) - 1] if ok else None
    success = round(100 * counts['200'] / n, 1) if n else None
    print(file.rsplit('/', 1)[-1], 'N=', n, dict(counts), 'success_%=', success, 'p95_200_ms=', p95)
METRICS
```

🔍 **Hisoblash formulalari va tahlil:**
* **Muvaffaqiyat ulushi (Success rate):**
  $$\text{Success \%} = \frac{N_{200}}{N_{\text{jami}}} \times 100\%$$
* **Rad etish ulushi (Rejection rate):**
  $$\text{Rejection \%} = \frac{N_{429}}{N_{\text{jami}}} \times 100\%$$
* **p95 kechikish (95th Percentile Latency):** Muvaffaqiyatli (200 OK) so‘rovlar kechikishining 95% qismi qaysi vaqtdan oshmaganini ko‘rsatadi.
  > [!CAUTION]
  > p95 faqat `status == 200` bo‘lgan so‘rovlardan hisoblanishi shart! 429 xatosi Nginx xotirasidan darhol (1-2 ms ichida) qaytadi. Agar 429 larni ham kechikish hisobiga qo‘shib yuborsak, tizim "hujum ostida tezlashib ketdi" degan soxta va xato xulosa kelib chiqadi.

#### Natijalar namunaviy qiyosiy jadvali:

| Bosqich / Mijoz | Jami so‘rov (N) | 200 OK | 429 Too Many | 5xx / ERR | p95 kechikish (200 OK), ms | Izoh |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **Oldin (before) / good** | 8 | 8 | 0 | 0 | ~205 ms | Normal mijoz to‘liq qoniqtirildi |
| **Oldin (before) / test** | 40 | 40 | 0 | 0 | ~220 ms | Kuzatuv rejimi: limit oshsa ham backendga o‘tdi |
| **Keyin (after) / good** | 8 | 8 | 0 | 0 | ~205 ms | **Eng muhim ko‘rsatkich: normal mijozga zarar yetmadi** |
| **Keyin (after) / test** | 40 | ~6 | ~34 | 0 | ~210 ms | Ortiqcha so‘rovlar 429 bilan bloklandi |

---

### 3.9. 8-bosqich: Xavfsizlikning Qo‘shimcha Tekshiruvlari

#### A) Backendni aylanib o‘tish (Bypass) sinovi
Tajovuzkor Nginx himoyasini chetlab o‘tib, backendga to‘g‘ridan-to‘g‘ri ulana oladimi?

```bash
ip netns exec a12-test curl --noproxy '*' --max-time 3 http://10.120.10.1:8080/
sleep 5
ip netns exec a12-good curl --noproxy '*' --max-time 3 -i http://10.120.10.1/?run=final
```

🔍 **Tahlil:**
* Birinchi buyruq xato berishi kerak (`Connection refused` yoki timeout). Chunki backend faqat `127.0.0.1:8080` da tinglaydi, `10.120.10.1:8080` da esa hech qanday xizmat yo‘q!
* Ikkinchi buyruq Nginx orqali (80-port) so‘rov yuboradi va muvaffaqiyatli `HTTP 200 OK` oladi. Bu backendning to‘g‘ri izolyatsiya qilinganini isbotlaydi.

#### B) Mijoz IP manzilini soxtalashtirish (X-Forwarded-For Spoofing) sinovi
Tajovuzkor soxta `X-Forwarded-For` yuborib limitni alday oladimi?

```bash
sleep 5
ip netns exec a12-test curl --noproxy '*' --max-time 3 -H 'X-Forwarded-For: 192.0.2.55' 'http://10.120.10.1/?run=xff'
tail -n 1 /opt/a12/access.jsonl
```

🔍 **Tahlil:**
* `access.jsonl` ning oxirgi qatorini ko‘rsak, `"ip":"10.120.10.12"` bo‘lib turadi (mijoz yozgan `192.0.2.55` emas!).
* Nega? Chunki Nginxda `real_ip` moduli yoqilmagan va har qanday begona mijoz uzatgan sarlavhaga ishonilmaydi. Nginx soket darajasidagi haqiqiy ulanish IPsini oladi.

---

# 4-QISM: SHARTLI ISHLAB CHIQARISH SSENARIYSI VA HISOB-KITOBLAR (6.1-BO‘LIM YECHIMLARI)

Hujjatning 6.1-bo‘limida quyidagi shartli parametrlar berilgan:
* **Portal uplinki (Internet kanali):** $100\text{ Mbit/s}$
* **Odatiy eng yuqori kiruvchi trafik (Peak traffic):** $40\text{ Mbit/s}$
* **Ilovaning barqaror ishlash sig‘imi (Capacity):** $200\text{ so‘rov/s}$ gacha
* **Tashqi tajovuzkor ssenariylari:**
  - Hajmli L3/L4 hujum: $1\text{ Gbit/s}$
  - Taqsimlangan L7 hujum: $1000\text{ ta manba}$, har biri $2\text{ so‘rov/s}$ dan yuboradi.
* **Qonuniy talabalar:** Bitta umumiy NAT IP manzili ortida $30\text{ ta talaba}$ bor; har biri $0.5\text{ so‘rov/s}$ yuboradi.

---

### 1-Savol: Kanal zaxirasini va 1 Gbit/s hujumining uplinkka nisbatini hisoblang.
* **Kanal zaxirasi (Bandwidth headroom):**
  $$\text{Zaxira} = \text{Uplink} - \text{Trafik} = 100\text{ Mbit/s} - 40\text{ Mbit/s} = \mathbf{60\text{ Mbit/s}}$$
* **1 Gbit/s ning nisbati:**
  $$1\text{ Gbit/s} = 1000\text{ Mbit/s}$$
  $$\text{Kattalik darajasi} = \frac{1000\text{ Mbit/s}}{100\text{ Mbit/s}} = \mathbf{10\text{ marta}}$$
* **Xulosa:** Hujum trafigi tashkilotning jismoniy internet kanalidan **10 barobar katta**. Bunday holatda serverdagi Nginx, iptables yoki firewall hech narsa qila olmaydi, chunki kanal serverga yetmasdanoq to‘lib bo‘lgan bo‘ladi. Buni faqat yuqori turuvchi Internet provayder (ISP) yoki trafikni tozalash markazi (Scrubbing Center) bartaraf eta oladi.

---

### 2-Savol: NAT ortidagi 30 talabaning jami tezligini hisoblang. Laboratoriyadagi 2r/s IP limiti ularni nega noto‘g‘ri cheklashini (False Positive) izohlang.
* **30 talabaning jami so‘rov tezligi:**
  $$\text{Jami tezlik} = 30 \times 0.5\text{ so‘rov/s} = \mathbf{15\text{ so‘rov/s}}$$
* **Nega 2r/s limiti noto‘g‘ri ishlaydi?**
  NAT (Network Address Translation) texnologiyasida yotoqxona, o‘quv binosi yoki tashkilotdagi barcha 30 nafar talaba internetga **bitta umumiy tashqi IP manzil** orqali chiqadi. Server bu 30 kishini bitta IP manzil deb ko‘radi.
  Agar serverda bir IP uchun `rate=2r/s` o‘rnatilgan bo‘lsa, talabalarning umumiy 15 so‘rov/s oqimidan atigi 2 tasi o‘tib, qolgan 13 tasi asossiz ravishda **429 Too Many Requests** olib bloklanadi (**False Positive — Soxta ijobiy rad etish**).
* **Yechim:** Limit faqat IP bo‘yicha emas, balki foydalanuvchi sessiyasi (Session Cookie, Bearer Token) yoki `limit_req` burst parametrini kattalashtirish orqali o‘rnatilishi lozim.

---

### 3-Savol: 1000 ta manbaning yig‘indi so‘rov tezligini hisoblang va 200 so‘rov/s ilova sig‘imi bilan solishtiring. Har IP o‘z limitidan oshmasa ham qanday muammo yuzaga keladi?
* **1000 ta manbaning yig‘indi so‘rov tezligi:**
  $$\text{Yig‘indi tezlik} = 1000 \times 2\text{ so‘rov/s} = \mathbf{2000\text{ so‘rov/s}}$$
* **Ilova sig‘imi bilan solishtirish:**
  $$\frac{2000\text{ so‘rov/s}}{200\text{ so‘rov/s}} = \mathbf{10\text{ barobar ortiq!}}$$
* **Muammo izohi (DDoSning mohiyati):**
  Har bir tajovuzkor bot alohida IPdan atigi 2 so‘rov/s yubormoqda, ya’ni serverdagi individual IP limitini (`rate=2r/s`) buzmayapti. Har bir IP nuqtai nazaridan so‘rov qonuniydek ko‘rinadi. Ammo ularning umumiy soni 1000 ta bo‘lgani uchun serverga 2000 so‘rov/s keladi. Server esa sekundiga ko‘pi bilan 200 ta so‘rovni bajara oladi.
  Oqibatda backend server protsessori va ma’lumotlar bazasi to‘liq ishdan chiqadi (crash bo‘ladi). Bu individual IP bo‘yicha cheklovlar taqsimlangan DDoS hujumlariga qarshi yetarli emasligini isbotlaydi.

---

### 4-Savol: Ilova uchun 20% zaxira qoldirilsa, global qabul chegarasining boshlang‘ich nomzodi nechaga teng?
* **Nomzod global chegara hisobi:**
  $$\text{Global limit} = 200\text{ so‘rov/s} \times (1 - 0.20) = 200 \times 0.8 = \mathbf{160\text{ so‘rov/s}}$$
* **Cheklov va amaliy izoh:**
  Ushbu 160 so‘rov/s chegarasi faqat umumiy boshlang‘ich taklifdir. Real tizimda har bir HTTP so‘rovining hisoblash "narxi" (Cost) har xil bo‘ladi. Masalan, keshlangan statik rasmni qaytarish uchun 160 so‘rov/s juda kam (server 5000 ta ham ko‘tara oladi), ammo og‘ir qidiruv yoki hisobot yaratuvchi SQL so‘rovi uchun hatto 50 so‘rov/s ham serverni qulatishi mumkin. Shuning uchun limitlar har bir API endpoint va resurs sarfiga qarab alohida kalibrlanishi shart.

---

# 5-QISM: VEB-PORTAL UCHUN DDOS HIMOYA LOYIHASI VA HODISAGA JAVOB REJASI (INCIDENT RESPONSE)

### 5.1. Qatlamli Himoya Rejasi (5 ta asosiy chora)

| Qatlam | Chora nomi | Qayerda ishlaydi? | Qanday signalda yoqiladi? | Kim javobgar? | Qaysi dalil bilan tekshiriladi? | Qachon bekor qilinadi (qaytariladi)? |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Provayder (L3/L4)** | Upstream BGP / Scrubbing (Trafikni tozalash) | Internet provayder yoki tashqi DDoS tozalash tarmog‘i | Kiruvchi kanal bandligi 80 Mbit/s dan (80%) oshib, 3 daqiqa davom etganda | Tarmoq muhandisi (Network Engineer) | Provayder port grafiki, kiruvchi paket/s va bit/s o‘lchovlari | Hujum to‘xtab, kanal bandligi 15 daqiqa davomida 40 Mbit/s dan past bo‘lganda |
| **2. Chegara (Edge/CDN)** | CDN va WAF qatlamida JS Challenge / Turnstile | CDN (Cloudflare, Akamai yoki Reverse Proxy klasteri) | Nginxda 5xx xatolari 5% dan oshganda yoki umumiy L7 so‘rovlar 500 r/s dan oshganda | Xavfsizlik bo‘yicha muhandis (SecOps) | WAF bloklash grafiki, CDN orqali o‘tgan qonuniy so‘rovlar nisbati | So‘rovlar oqimi me’yoriy darajaga tushib, bot faolligi to‘xtaganda |
| **3. Server / Proxy (L7)** | Dynamic Rate Limiting va Global Limit | Nginx Reverse Proxy (`/opt/a12/nginx.conf`) | Backend CPU bandligi 85% dan oshganda yoki p95 kechikishi 1 soniyadan oshganda | Tizim administratori (SysAdmin) | Nginx `access.jsonl` loglari, 429 status kodi ulushi | CPU yuklamasi 50% dan pasayganda va p95 me’yorga (250 ms) qaytganda |
| **4. Ilova va Baza** | Og‘ir qidiruv/hisobot so‘rovlarini vaqtincha cheklash, Keshni yoqish | Veb-ilova kodi (Django, Node.js va h.k.) va Redis kesh | Ma’lumotlar bazasi ulanishlar puli (Connection pool) 90% ga to‘lganda | Backend dasturchi (DevOps/Lead Dev) | DB `active connections` ko‘rsatkichi, Redis `hit-ratio` | Baza yuklamasi me’yoriy darajaga tushgach |
| **5. Monitoring** | Real vaqtda metrikalar tahlili va ogohlantirish | Prometheus, Grafana, Alertmanager | Normal mijoz muvaffaqiyati 95% dan tushganda yoki p95 800 ms dan oshganda | Navbatchi muhandis (SRE / On-call) | Grafana Dashboard, SLO (Service Level Objectives) metrikalari | Tizimning barqaror holati to‘liq tiklanganda |

---

### 5.2. Hodisaga Javob Berish Rejasi (Incident Response Timeline)

* **0 – 5 daqiqa (Dastlabki aniqlash):**
  - Monitoringdan signal keladi (masalan, p95 oshdi yoki normal mijoz muvaffaqiyati 95% dan pasaydi).
  - Navbatchi administrator oddiy mijoz nomidan xizmatni tekshiradi (`curl`).
  - Nosozlik sababi ajratiladi: bu ichki dasturiy xatomi (masalan, yangi noto‘g‘ri deploy) yoki tashqi trafik hujumimi?
* **5 – 10 daqiqa (Tarmoq eskalatsiyasi):**
  - Tarmoq mutaxassisi kanal bandligini tekshiradi.
  - Agar tashqi 100 Mbit/s kanal to‘lgan bo‘lsa, zudlik bilan Internet provayderga (ISP) qo‘ng‘iroq qilinadi va trafikni tozalash (Scrubbing / Blackholing) yoqiladi.
* **5 – 15 daqiqa (L7 himoyasini faollashtirish):**
  - Ilova va proxy administratori oldindan sinovdan o‘tgan Nginx rate limiting va global limit qoidalarini faol rejimga o‘tkazadi (`nginx -s reload`).
  - Oddiy foydalanuvchilar holati kuzatiladi (ular 429 olmasligi kerak).
* **Barqarorlashgach (Recovery):**
  - Tizim 10–15 daqiqa barqaror ishlagach, hodisa koordinatori ruxsati bilan favqulodda cheklovlar bosqichma-bosqich yumshatiladi.
* **Hodisa yakuni (Post-Incident Review):**
  - Barcha loglar, CSV dalillar saqlanadi.
  - Xizmat to‘xtash vaqti (Downtime) va yo‘qotishlar hisoblanadi.
  - Himoya qoidalariga tuzatishlar kiritiladi.

---

# 6-QISM: O‘QITUVCHI UCHUN HIMOYA SAVOLLARI VA MUKAMMAL JAVOBLAR (12 TA SAVOL)

Laboratoriya ishini o‘qituvchiga topshirishda berilishi mumkin bo‘lgan 12 ta nazorat savollarining to‘liq va asosli javoblari:

#### 1. DoS va DDoSning farqi nimada? Biz bajargan tajribada ko‘p manbali DDoS o‘lchandimi?
* **Javob:** DoS (Denial of Service) bitta manbadan bitta nishonga uyushtiriladi. DDoS (Distributed DoS) esa ko‘plab tarqalgan manbalardan (botnet) bir vaqtda amalga oshiriladi.
* Biz bajargan laboratoriya tajribasida ko‘p manbali haqiqiy DDoS **o‘lchanmadi**. Unda bitta tezkor mijoz (`a12-test`) va bitta normal mijoz (`a12-good`) ishtirok etdi. Bu kichik nazoratli tajriba bo‘lib, Nginxning L7 cheklov mexanizmini tekshirishga xizmat qiladi. Ko‘p manbali holat 6.1-bo‘limdagi hisob-kitoblarimizda nazariy va matematik jihatdan tahlil qilindi.

#### 2. Nima uchun backendga 127.0.0.1:8080 berilgan? Faqat 80-portdagi limit originni o‘zi himoyalay oladimi?
* **Javob:** Backend faqat `127.0.0.1:8080` (loopback) interfeysida tinglaydi, ya’ni tashqi tarmoqqa yopiq. Agar backend `0.0.0.0:8080` da tinglasa, tajovuzkor Nginx turgan 80-portni chetlab o‘tib (bypass), to‘g‘ridan-to‘g‘ri `http://SERVER_IP:8080` ga ulanadi va Nginxning hech qanday rate limiti ish bermaydi. Shuning uchun origin server faqat Nginx orqali so‘rov qabul qiladigan qilib izolyatsiya qilinishi shart.

#### 3. dry_run rejimida REJECTED_DRY_RUN bilan up=200 birga chiqishi nimani anglatadi?
* **Javob:** Bu Nginxning kuzatuv rejimida ishlayotganini anglatadi. `REJECTED_DRY_RUN` yozuvi so‘rov belgilangan limitdan oshganini va faol rejimda rad etilishi kerakligini bildiradi. Ammo `dry_run on` bo‘lgani uchun Nginx so‘rovni bloklamasdan backendga o‘tkazgan va backend muvaffaqiyatli ishlab, `up=200` (upstream status 200) javobini qaytargan.

#### 4. burst=4 va nodelay parametrlarini vaqt bo‘yicha izohlang. 2r/s qiymati bir zumdagi barcha oqimni ikkiga tushiradimi?
* **Javob:** `rate=2r/s` degani har 500 ms da bitta so‘rov o‘tadi deganidir. Agar bir zumda 6 ta so‘rov kelsa:
  - 1-so‘rov zudlik bilan o‘tadi;
  - Keyingi 4 ta so‘rov `burst=4` tufayli qabul qilinadi;
  - `nodelay` parametri bo‘lgani uchun bu 4 ta so‘rov sun’iy ravishda navbatda kuttirilmasdan, zudlik bilan parallel backendga yuboriladi;
  - 6-so‘rov esa burst chegarasidan toshib ketgani uchun darhol 429 xatosi bilan rad etiladi.
  Demak, 2r/s qiymati bir zumdagi oqimni birdaniga ikkiga tushirmaydi, balki burst doirasidagi qisqa portlashlarga ruxsat beradi.

#### 5. Nima uchun oddiy mijoz alohida IPdan va tezkor mijoz bilan bir vaqtda tekshiriladi?
* **Javob:** Xavfsizlik tizimining sifat mezoni faqat tajovuzkorni bloklash emas, balki **tajovuzkorni bloklash paytida oddiy qonuniy foydalanuvchiga zarar yetkazmaslikdir (No Collateral Damage)**. Ular bir vaqtda parallel yuborilganda, tezkor mijoz 429 olib bloklanayotgan bir paytda oddiy mijoz o‘zining 200 javoblarini to‘siqsiz olayotgani isbotlanadi.

#### 6. HTTP 429 tez javob berishi tizimning foydalanuvchilar uchun tezlashganini anglatadimi?
* **Javob:** **Yo‘q, mutlaqo unday emas!** 429 xatosi Nginx xotirasidan 1-2 millisekundda qaytadi. Bu so‘rov rad etilganini bildiradi, xizmat ko‘rsatilganini emas. Agar p95 kechikish hisobiga 429 larni qo‘shsak, o‘rtacha vaqt sun’iy ravishda kamayib, soxta yaxshilanish illyuziyasini yaratadi. Shuning uchun foydalanuvchi qoniqishini faqat muvaffaqiyatli (200 OK) so‘rovlar kechikishi bilan o‘lchash kerak.

#### 7. Nima uchun mijoz yuborgan X-Forwarded-For sarlavhasiga avtomatik ishonish mumkin emas?
* **Javob:** Chunki HTTP sarlavhalarini ixtiyoriy mijoz osongina soxtalashtirishi (Spoofing) mumkin. Agar server har qanday `X-Forwarded-For` ga ishonsa, tajovuzkor har bir so‘rovda har xil IP yozib (`X-Forwarded-For: 1.1.1.1`, `1.1.1.2`...) bitta IPga qo‘yilgan cheklovni bemalol chetlab o‘tadi. Bunga faqat server oldida turgan ishonchli proksi (masalan, Cloudflare) IP diapazonlari ro‘yxati kiritilgandagina ruxsat beriladi.

#### 8. NAT ortidagi 30 talaba uchun IP bo‘yicha 2r/s limitning ta’siri qanday?
* **Javob:** 30 talaba bitta umumiy NAT IP orqali sekundiga jami $30 \times 0.5 = 15\text{ so‘rov}$ yuboradi. Agar server har bir IPga 2 r/s cheklov qo‘ysa, bu 30 talabaning 13 ta so‘rovi asossiz rad etiladi (False Positive). Natijada darsdagi talabalar portalga kira olmay qoladi.

#### 9. 100 Mbit/s kanal 1 Gbit/s kiruvchi trafik bilan to‘lsa, lokal Nginx nimani hal qila olmaydi?
* **Javob:** Nginx server ichida ishlaydi. 1 Gbit/s trafik esa serverga kelayotgan jismoniy 100 Mbit/s tarmoq kabelini (uplink) to‘ldirib tashlaydi. Paketlar server tarmoq kartasiga yetib kelmasdanoq yo‘lda (provayder routerida) tashlab yuboriladi (drop bo‘ladi). Nginx qanchalik optimal sozlanmasin, o‘ziga yetib kelmagan paketni boshqara olmaydi. Buni faqat provayder darajasida filtrlash mumkin.

#### 10. Global so‘rov limiti backendni saqlasa ham qonuniy mijozlar nima sababdan rad etilishi mumkin?
* **Javob:** Global limit (masalan, butun server uchun 160 r/s) backendning qulashini oldini oladi. Biroq, agar tajovuzkor botnet birdaniga 1000 ta so‘rov yuborib, ushbu 160 ta limit kvotasini to‘liq egallab olsa, qonuniy oddiy foydalanuvchilar uchun bo‘sh joy qolmaydi va ular ham 429 xatosi bilan rad etiladi (**Denial of Service to legitimate users**).

#### 11. Normal foydalanuvchi mavjudligini qaysi metrikalar bilan isbotlaysiz va qaysi signalda siyosatni qaytarasiz?
* **Javob:**
  - **Isbotlovchi metrikalar:** Normal mijoz so‘rovlarining muvaffaqiyat ulushi ($\ge 95\%$), 5xx server xatolarining yo‘qligi va 200 OK javoblarining p95 kechikishi ($< 500\text{ ms}$).
  - **Siyosatni qaytarish (Rollback) signali:** Agar himoya yoqilgach, normal mijozlarning 429 xatolari 5% dan oshsa yoki xizmatdan foydalanish darajasi keskin tushib ketsa, yangi joriy etilgan qat’iy siyosat darhol bekor qilinadi.

#### 12. Ushbu laboratoriyadan olingan natijalarni real portal sig‘imiga ko‘chirish uchun yana qanday o‘lchovlar kerak?
* **Javob:**
  - Har xil og‘irlikdagi API endpointlar resurs sarfi (Profiling);
  - Ma’lumotlar bazasining maksimal tranzaksiyalar sig‘imi (TPS / IOPS);
  - SSL/TLS handshakelarining CPUga keltiradigan yuki;
  - Haqiqiy foydalanuvchilarning sessiya vaqtlari va cookie orqali autentifikatsiya qilingan trafigi;
  - Statik kontentning CDN keshlanish darajasi (Cache hit ratio).

---

# 7-QISM: NOSOZLIKLARNI BARTARAF ETISH (TROUBLESHOOTING) VA LABNI TOZALASH

### 7.1. Nosozliklar Jadvali

| Muammo / Xatolik belgisi | Tekshirish usuli | Sababi va Tuzatish yo‘li |
| :--- | :--- | :--- |
| **`RTNETLINK answers: File exists`** | `ip link show br-a12`, `ip netns list` | Tarmoq interfeyslari yoki namespace avvaldan mavjud. 7.3-bo‘limdagi tozalash buyruqlarini bajaring yoki toza snapshotga qayting. |
| **`nginx: configuration file test failed`** | `nginx -t -c /opt/a12/nginx.conf` | Konfiguratsiyada sintaksis xatosi (qavs yopilmagan yoki `;` nuqtali vergul tushib qolgan). Chiqqan satr raqamini tekshirib tuzating. |
| **HTTP 502 Bad Gateway** | `systemctl status a12-backend`, `ss -lntp` | Backend ishlamayapti yoki Nginx boshqa portga murojaat qilmoqda. `systemctl restart a12-backend` qiling va port `127.0.0.1:8080` ekanini tekshiring. |
| **HTTP 503 xatosi kutilmaganda chiqsa** | `cat /opt/a12/error.log` | Nginxda `limit_req_status 429;` ko‘rsatilmagan bo‘lsa standart 503 qaytadi. Faylni tekshiring. |
| **`after` bosqichida 429 chiqmasa** | `grep dry_run /opt/a12/nginx.conf` | `dry_run off` qilinmagan yoki Nginx qayta yuklanmagan (`nginx -s reload`). |
| **Oddiy mijoz (`good`) ham 429 olsa** | `cat /opt/a12/results/good-after.csv` | Sinovlar orasida 5 soniyalik tanaffus qilinmagan yoki mijozlar bitta IPga tushib qolgan. |

### 7.2. Konfiguratsiyani Orqaga Qaytarish (Rollback)
Agar faol cheklov paytida muammo yuzaga kelsa, zudlik bilan avvalgi kuzatuv rejimiga qaytiladi:

```bash
cp /opt/a12/nginx-before.conf /opt/a12/nginx.conf
nginx -t -c /opt/a12/nginx.conf && nginx -s reload -c /opt/a12/nginx.conf
```

### 7.3. Dalillarni Saqlash va Laboratoriyani Tozalash
Laboratoriya tugagach, hisobot uchun natijalarni (`/opt/a12/results/*.csv`, `access.jsonl`, `error.log`) nusxalab oling.

Resurslarni to‘liq tozalash buyruqlari:

```bash
nginx -s quit -c /opt/a12/nginx.conf
systemctl stop a12-backend
systemctl disable a12-backend
rm /etc/systemd/system/a12-backend.service
systemctl daemon-reload
ip netns del a12-good
ip netns del a12-test
ip link del br-a12
```

🔍 **Tozalash buyruqlari vazifasi:**
* `nginx -s quit ...`: Ishlayotgan maxsus Nginx jarayonini to‘xtatadi.
* `systemctl stop/disable ...`: Python backend xizmatini to‘xtatadi va o‘chiradi.
* `rm ... && systemctl daemon-reload`: Yaratilgan tizim servisini o‘chirib, tizimni yangilaydi.
* `ip netns del ...`: Yaratilgan virtual mijozlar muhitini o‘chiradi.
* `ip link del br-a12`: Virtual tarmoq ko‘prigini o‘chiradi (unga ulangan barcha veth kabellar ham avtomatik o‘chiriladi).

---

# 8-QISM: FOYDALANILGAN RASMIY VA ILMIY MANBALAR

1. **NCSC (National Cyber Security Centre, UK):** *Denial of Service (DoS) guidance collection*.
   - Havola: [https://www.ncsc.gov.uk/collection/denial-service-dos-guidance-collection](https://www.ncsc.gov.uk/collection/denial-service-dos-guidance-collection)
   - DoS/DDoS hujumlariga tayyorgarlik, xizmat arxitekturasini tushunish va hodisalarga javob berish bo‘yicha milliy standart qo‘llanma.
2. **Nginx Documentation:** *Module ngx_http_limit_req_module*.
   - Havola: [https://nginx.org/en/docs/http/ngx_http_limit_req_module.html](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
   - Leaky bucket algoritmi, `limit_req_zone`, `burst`, `nodelay` va `limit_req_dry_run` parametrlarining rasmiy spetsifikatsiyasi.
3. **NGINX Admin Guide:** *Limiting Access to Proxied HTTP Resources*.
   - Havola: [https://docs.nginx.com/nginx/admin-guide/security-controls/controlling-access-proxied-http/](https://docs.nginx.com/nginx/admin-guide/security-controls/controlling-access-proxied-http/)
   - Proksi orqali uzatiluvchi resurslar uchun xavfsizlik va so‘rov tezligini cheklash qo‘llanmasi.
4. **Nginx Documentation:** *Module ngx_http_proxy_module*.
   - Havola: [https://nginx.org/en/docs/http/ngx_http_proxy_module.html](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
   - Reverse proxy sozlamalari, `proxy_pass`, ulanish va kutish timeoutlari (`proxy_read_timeout`).
5. **Nginx Documentation:** *Module ngx_http_log_module*.
   - Havola: [https://nginx.org/en/docs/http/ngx_http_log_module.html](https://nginx.org/en/docs/http/ngx_http_log_module.html)
   - `log_format` direktivasi, JSON formatida log yozish va o‘zgaruvchilar ro‘yxati.
6. **Nginx Documentation:** *Module ngx_http_realip_module*.
   - Havola: [https://nginx.org/en/docs/http/ngx_http_realip_module.html](https://nginx.org/en/docs/http/ngx_http_realip_module.html)
   - Proksi ortidagi haqiqiy mijoz IP manzilini xavfsiz aniqlash va ishonchli manbalar ro‘yxatini tuzish.
7. **IETF RFC 6585:** *Additional HTTP Status Codes — Section 4: 429 Too Many Requests*.
   - Havola: [https://datatracker.ietf.org/doc/html/rfc6585#section-4](https://datatracker.ietf.org/doc/html/rfc6585#section-4)
   - Tezlik cheklovi tufayli yuzaga keladigan 429 maqom kodining xalqaro standarti.
