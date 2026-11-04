# TARMOQ XAVFSIZLIGI — A12 AMALIY MASHG‘ULOT
## LABORATORIYA ISHI BO‘YICHA YAKUNIY TAHLIL, QO‘LLANILGAN VOSITALAR VA KASBIY RIVOJLANISH QO‘LLANMASI
### Hujjat: `conclusion.md`

---

## 📌 KIRISH
Ushbu hujjat **A12 amaliy mashg‘uloti** («DDoS hujumlaridan himoyalanish choralarini loyihalash: Veb-xizmat uchun qatlamli himoya va nazoratli sinov») davomida bajarilgan barcha ishlarning chuqur xulosasi, qo‘llanilgan vosita va kutubxonalar tahlili, CLI buyruqlar katalogi, shuningdek, kelgusida mustaqil ta’lim (**self-study**) uchun professional yo‘l xaritasi (**roadmap**) va shaxsiy amaliy tavsiyalarni o‘z ichiga oladi.

---

# 1-BO‘LIM: VAZIFANI ASOSIY BAJARISHDAN MAQSAD (NIMA UCHUN BU ISH QILINDI?)

Axborot xavfsizligida klassik **CIA triadasining** eng muhim komponentlaridan biri — **Mavjudlik (Availability)** hisoblanadi. Axborot qanchalik maxfiy va butun saqlanmasin, agar foydalanuvchi kerakli paytda tizimga kira olmasa, xizmat o‘z qiymatini yo‘qotadi.

Ushbu amaliy vazifani bajarishdan ko‘zlangan **asosiy maqsadlar**:

1. **L7 (Application Layer) darajasidagi xizmatdan foydalanishni rad etish (DoS/DDoS) mexanizmlarini tushunish:**
   Ko‘pincha xavfsizlik deganda faqat tarmoqlararo ekran (Firewall) yoki portlarni yopish tushuniladi. Biroq, zamonaviy DDoS hujumlarining katta qismi 80 (HTTP) va 443 (HTTPS) ochiq portlariga qonuniy ko‘rinishdagi so‘rovlar oqimini yuborish orqali amalga oshiriladi. Vazifaning asosiy maqsadi — ilova darajasidagi ushbu tahdidlarni aniqlash va ularni cheklashni o‘rganish.

2. **Reverse Proxy (Nginx) yordamida tezlikni nazoratli cheklash (Rate Limiting):**
   Veb-server oldiga Nginx teskari proksisini o‘rnatib, **Leaky Bucket** algoritmi asosida so‘rovlar sonini me’yorlashtirish (`rate=2r/s`, `burst=4`, `nodelay`) amaliyotini o‘zlashtirish.

3. **Nazoratli va xavfsiz sinov (Safe Benchmarking) madaniyatini shakllantirish:**
   Haqiqiy universitet yoki tashkilot tarmog‘iga zarar yetkazmasdan, bitta Ubuntu Server VM ichida Linux Network Namespaces texnologiyasi yordamida izolyatsiyalangan **qonuniy mijoz (`a12-good`)** va **tajovuzkor yuklama (`a12-test`)** oqimlarini hosil qilish hamda ularning to‘qnashuvini laboratoriya sharoitida modellashtirish.

4. **Siyosatni xavfsiz joriy qilish metodologiyasi (Dry-run -> Active):**
   Ishlab chiqarish (production) muhitida yangi xavfsizlik qoidasini birdaniga yoqish qonuniy foydalanuvchilarni bloklab qo‘yishi mumkin (**False Positive**). Laboratoriya orqali avval **kuzatuv rejimi (`dry_run on`)** yordamida qoida ta’siri o‘rganildi, so‘ngra **faol cheklov (`dry_run off`)** rejimiga o‘tkazildi.

5. **Qatlamli himoya (Defense-in-Depth) va chegaralarni anglash:**
   Mahalliy Nginx rate limiting barcha muammolarni hal qila olmasligini (masalan, $1\text{ Gbit/s}$ hajmli to‘fon yoki NAT ortidagi yuzlab talabalar muammosini) matematik hisob-kitoblar va amaliy sinovlar orqali isbotlash.

---

# 2-BO‘LIM: QO‘LLANILGAN TASHQI KUTUBXONALAR, VOSITALAR (TOOLS) VA ULARNING VAZIFALARI

Ushbu laboratoriya ishida hech qanday uchinchi tomon noxolis yoki xavfli kutubxonalari ishlatilmadi. Faqat sanoat standarti hisoblangan rasmiy, barqaror va xavfsiz Linux utilitalari hamda Python standart kutubxonalari qo‘llanildi:

```
+---------------------------------------------------------------------------------------+
|                                 QO‘LLANILGAN VOSITALAR                                |
+---------------------------------------------------------------------------------------+
|  1. TARMOQ VA VIRTUALIZATSIYA    |  2. VEB-SERVER VA PROKSI   |  3. DASTURLASH TILI   |
|  - iproute2 (ip link, ip netns)  |  - Nginx Open Source (OSS) |  - Python 3 (Standart)|
|  - Linux Bridge (br-a12)         |  - ngx_http_limit_req_mod  |    * http.server      |
|  - Virtual Ethernet (veth)       |  - ngx_http_proxy_mod      |    * concurrent.fut   |
|  - systemd (Service management)  |  - ngx_http_log_mod (JSON) |    * urllib.request   |
|  - ss (Socket Statistics)        |                            |    * csv, math, time  |
|  - curl (HTTP Client CLI)        |                            |                       |
+---------------------------------------------------------------------------------------+
```

### 1. Tizim va Tarmoq Vositalari:
* **`iproute2` to‘plami (`ip` buyrug‘i):**
  - Zamonaviy Linux tizimlarida tarmoqni boshqarish uchun asosiy vosita (eski `ifconfig`, `route` o‘rniga kelgan).
  - `ip netns`: Tarmoq nomlar fazosi (Network Namespace) yaratish — bitta operatsion tizim ichida bir-biridan mustaqil tarmoq steklarini shakllantirish uchun xizmat qildi.
  - `ip link` va `veth`: Virtual tarmoq kartalari va "virtual patch-kordlar" (veth juftliklari) yaratish uchun qo‘llanildi.
  - `bridge`: Virtual kommutator (switch) vazifasini bajaruvchi ko‘prik (`br-a12`) yaratishda foydalanildi.
* **`systemd` va `systemctl`:**
  - Linux tizim va xizmatlar menejeri (Init system).
  - Biz yaratgan Python backend dasturini doimiy ishlovchi, xavfsiz va nazorat qilinadigan tizim xizmati (`a12-backend.service`) sifatida ro‘yxatdan o‘tkazish va boshqarish uchun ishlatildi.
* **`ss` (Socket Statistics):**
  - Tarmoq soketlari, ochiq portlar va ularni tinglayotgan jarayonlarni ko‘rsatuvchi utilita (`netstat` ning zamonaviy analogi).
  - Portlar faqat belgilangan interfeyslarda tinglayotganini (masalan, backend faqat `127.0.0.1:8080` da, Nginx esa `10.120.10.1:80` da) tekshirish uchun qo‘llanildi.
* **`curl`:**
  - Buyruqlar satri orqali URL manzillar bilan ma’lumot almashish vositasi.
  - Tizimni qo‘lda test qilish, maxsus HTTP sarlavhalarini (`X-Forwarded-For`) soxtalashtirib yuborish va backendni aylanib o‘tish (bypass) sinovlarida foydalanildi.
* **Matn utilitalari (`sed`, `grep`, `nano`, `cat`):**
  - Konfiguratsiya fayllarini yaratish (`cat <<'EOF'`), satrlarni avtomatik o‘zgartirish (`sed -i`), loglardan kerakli qatorlarni filtrlab olish (`grep 'REJECTED'`) uchun qo‘llanildi.

### 2. Veb-Server va Proksi Vositalari:
* **`Nginx OSS` (Open Source):**
  - Yuqori unumdorlikka ega teskari proksi (Reverse Proxy) va HTTP server.
  - **`ngx_http_limit_req_module`:** Xotirada umumiy zona (`limit_req_zone`) ochib, Leaky Bucket algoritmi bo‘yicha so‘rovlarni cheklash uchun ishlatildi.
  - **`ngx_http_proxy_module`:** Mijoz so‘rovlarini orqa fondagi Python backendga yo‘naltirish (`proxy_pass`) va ulanish muddatlarini cheklash (`timeout`) uchun qo‘llanildi.
  - **`ngx_http_log_module`:** Barcha kiruvchi so‘rovlar natijasini tahlil qilish uchun strukturalangan **JSON formatida (`access.jsonl`)** log yozishda ishlatildi.

### 3. Dasturlash Tili va Standart Kutubxonalari (Python 3):
Hech qanday qo‘shimcha `pip install` qilmasdan, to‘g‘ridan-to‘g‘ri Pythonning o‘zida mavjud bo‘lgan quyidagi modullar ishlatildi:
* **`http.server` (`BaseHTTPRequestHandler`, `ThreadingHTTPServer`):**
  - Haqiqiy veb-xizmat vazifasini bajaruvchi, har bir so‘rovga 200 ms sun’iy kechikish bilan javob qaytaruvchi ko‘p oqimli test backendini yaratishda foydalanildi.
* **`concurrent.futures` (`ThreadPoolExecutor`):**
  - Bir vaqtning o‘zida parallel 4 ta ishchi oqim (thread) orqali 40 ta HTTP so‘rovini tezkor yuboruvchi tajovuzkor yuklama generatorini yaratishda ishlatildi.
* **`urllib.request` (`build_opener`, `ProxyHandler`, `Request`, `HTTPError`):**
  - Linux global proksi sozlamalarini chetlab o‘tib (`ProxyHandler({})`), to‘g‘ridan-to‘g‘ri lokal virtual IPga so‘rov yuborish va HTTP status kodlarini (200, 429) aniqlashda foydalanildi.
* **`time` (`time.monotonic()`, `time.sleep()`):**
  - So‘rovlar orasida 1 soniyalik oraliq saqlash (oddiy mijozda) va har bir so‘rovning bajarilish vaqtini millisekundlarda o‘ta aniq o‘lchashda ishlatildi.
* **`csv` moduli:**
  - Olingan barcha natijalarni (so‘rov IDsi, status kodi, sarflangan vaqt) keyinchalik tahlil qilish uchun `.csv` jadvallariga saqlashda ishlatildi.
* **`collections.Counter` va `math`:**
  - Sinov tugagach, to‘rtta CSV fayl bo‘yicha 200, 429, 5xx xatolar sonini, muvaffaqiyat foizini va **p95 kechikish** ko‘rsatkichini hisoblashda foydalanildi.

---

# 3-BO‘LIM: HAR BIR TOOL YORDAMIDA BERILGAN KETMA-KET CLI BUYRUQLAR VA ULARNING MAQSADI

Laboratoriya davomida berilgan barcha buyruqlar ularning mantiqiy maqsadi bilan quyidagi katalogda jamlandi:

### 1-Bosqich: Tizim Paketlari va Muhitni Tayyorlash
| CLI Buyruq | Qo‘llanilgan Tool | Bajarilish Maqsadi |
| :--- | :--- | :--- |
| `sudo apt update` | APT (Advanced Package Tool) | Repozitoriyadagi paketlar indeksini yangilash. |
| `sudo apt install -y nginx python3 iproute2 curl nano` | APT | Laboratoriya uchun talab etiladigan barcha dasturiy vositalarni internetdan yuklab o‘rnatish. |
| `sudo systemctl stop nginx` | systemd | Tizim bilan birga fon rejimida avtomatik ishga tushgan standart Nginx xizmatini to‘xtatish (alohida nusxa ishlatish uchun). |
| `nginx -v` | Nginx | O‘rnatilgan Nginx versiyasini tekshirish. |
| `python3 --version` | Python | Python interpretatori mavjudligini tasdiqlash. |

### 2-Bosqich: Virtual Tarmoq Infratuzilmasini Qurish
| CLI Buyruq | Qo‘llanilgan Tool | Bajarilish Maqsadi |
| :--- | :--- | :--- |
| `sudo -i` | Bash / Sudo | Keyingi barcha tarmoq amallarini root huquqi bilan bajarish uchun xavfsiz sessiya ochish. |
| `mkdir -p /opt/a12/results` | Coreutils (`mkdir`) | Laboratoriya natijalari (CSV, loglar) saqlanadigan kataloglar ierarxiyasini yaratish. |
| `ip link add br-a12 type bridge` | iproute2 (`ip link`) | Linux yadrosida `br-a12` nomli virtual tarmoq ko‘prigini (L2 kommutator) yaratish. |
| `ip addr add 10.120.10.1/24 dev br-a12` | iproute2 (`ip addr`) | Virtual ko‘prikka `10.120.10.1` manzilini biriktirish (bu Nginx uchun shlyuz va kirish nuqtasi bo‘ladi). |
| `ip link set br-a12 up` | iproute2 (`ip link`) | Virtual ko‘prik interfeysini faol (UP) holatga o‘tkazish. |
| `ip netns add a12-good` | iproute2 (`ip netns`) | Qonuniy oddiy mijoz uchun alohida tarmoq steki (Network Namespace) yaratish. |
| `ip netns add a12-test` | iproute2 (`ip netns`) | Tajovuzkor/tezkor mijoz uchun ikkinchi alohida tarmoq stekini yaratish. |
| `ip link add v-good type veth peer name c-good` | iproute2 (`ip link`) | Oddiy mijoz uchun 2 uchli virtual Ethernet kabel juftligini (`v-good` va `c-good`) yaratish. |
| `ip link set v-good master br-a12` | iproute2 (`ip link`) | Virtual kabelning bir uchini (`v-good`) `br-a12` ko‘prigiga ulash. |
| `ip link set v-good up` | iproute2 (`ip link`) | Ko‘prikka ulangan portni faollashtirish. |
| `ip link set c-good netns a12-good` | iproute2 (`ip link`) | Virtual kabelning ikkinchi uchini (`c-good`) `a12-good` namespace'i ichiga kiritish. |
| `ip -n a12-good link set lo up` | iproute2 (`ip`) | Namespace ichidagi mahalliy loopback interfeysini yoqish. |
| `ip -n a12-good link set c-good up` | iproute2 (`ip`) | Namespace ichidagi mijoz interfeysini faollashtirish. |
| `ip -n a12-good addr add 10.120.10.11/24 dev c-good` | iproute2 (`ip`) | Oddiy mijozga `10.120.10.11` IP manzilini biriktirish. |
| *(Xuddi shu amallar `test:12` uchun bajarildi)* | iproute2 | Tezkor mijozga `10.120.10.12` IP manzilini biriktirish. |
| `ip -br addr show br-a12` | iproute2 | Ko‘prikning IP manzili va holatini qisqa jadval ko‘rinishida tekshirish. |
| `ip -n a12-good route` | iproute2 (`ip route`) | Namespace ichida default gateway yo‘qligini va faqat 10.120.10.0/24 tarmog‘i mavjudligini tasdiqlash. |

### 3-Bosqich: Backend Xizmatini Yaratish va Ishga Tushirish
| CLI Buyruq | Qo‘llanilgan Tool | Bajarilish Maqsadi |
| :--- | :--- | :--- |
| `cat > /opt/a12/backend.py <<'APP' ... APP` | Bash Here-Doc | Test backend serverining Python kodini faylga yozish. |
| `chmod 644 /opt/a12/backend.py` | Coreutils (`chmod`) | Faylga xavfsiz o‘qish huquqini berish. |
| `cat > /etc/systemd/system/a12-backend.service ...` | Bash Here-Doc | Python backendni doimiy fon xizmati qiluvchi systemd unit konfiguratsiyasini yaratish. |
| `systemctl daemon-reload` | systemd | Yangi yaratilgan servis faylini tizimga tanitish. |
| `systemctl start a12-backend` | systemd | Backend xizmatini ishga tushirish. |
| `systemctl is-active a12-backend` | systemd | Xizmat faol (`active`) holatda ekanligini tekshirish. |
| `curl --noproxy '*' --max-time 3 http://127.0.0.1:8080/` | curl | Backend mahalliy soketda ishlayotganini bevosita HTTP so‘rovi bilan tekshirish. |
| `ss -lntp 'sport = :8080'` | Socket Statistics (`ss`) | 8080-port faqat va faqat `127.0.0.1` (loopback)da tinglayotganini tekshirish. |

### 4-Bosqich: Nginx Konfiguratsiyasini Yaratish va Ishga Tushirish
| CLI Buyruq | Qo‘llanilgan Tool | Bajarilish Maqsadi |
| :--- | :--- | :--- |
| `cat > /opt/a12/nginx.conf <<'CONF' ... CONF` | Bash Here-Doc | JSON log, `limit_req_zone`, `dry_run on` va proxy parametrlariga ega to‘liq Nginx konfiguratsiyasini yaratish. |
| `nginx -t -c /opt/a12/nginx.conf` | Nginx | Yaratilgan konfiguratsiyaning sintaksis xatolarini tekshirish. |
| `nginx -c /opt/a12/nginx.conf` | Nginx | Nginxni standart yo‘ldan emas, aynan `/opt/a12/nginx.conf` fayli bilan alohida jarayon sifatida ishga tushirish. |
| `ip netns exec a12-good curl --noproxy '*' -i http://10.120.10.1/?run=setup` | iproute2 + curl | `a12-good` namespace ichidan Nginx orqali backendga ilk sinov so‘rovini yo‘llash. |
| `ss -lntp 'sport = :80'` | Socket Statistics (`ss`) | Nginx `10.120.10.1:80` da so‘rov kutayotganini tekshirish. |

### 5-Bosqich: Nazoratli Sinov Dasturi va "Oldin" (Before) Bosqichi
| CLI Buyruq | Qo‘llanilgan Tool | Bajarilish Maqsadi |
| :--- | :--- | :--- |
| `cat > /opt/a12/probe.py <<'PROBE' ... PROBE` | Bash Here-Doc | Normal (8 ta) va Fast (40 ta parallel) so‘rov yuboruvchi Python probasini yaratish. |
| `python3 -m py_compile /opt/a12/backend.py /opt/a12/probe.py` | Python py_compile | Yozilgan Python skriptlarini baytkodga kompilyatsiya qilib, sintaktik tozaligini tasdiqlash. |
| `sleep 5` | Coreutils (`sleep`) | Rate limit hisoblagichlari nollanishi uchun 5 soniya kutish. |
| `timeout 45s ip netns exec a12-good python3 ... normal before ... &` | timeout + netns + Python | Oddiy mijoz nomidan 8 ta so‘rovni fonda (`&`) ishga tushirish. |
| `GOOD_PID=$!` | Bash | Fondagi jarayon ID raqamini o‘zgaruvchiga saqlash. |
| `timeout 45s ip netns exec a12-test python3 ... fast before ...` | timeout + netns + Python | Tezkor mijoz nomidan 40 ta parallel so‘rovni shu vaqtning o‘zida ishga tushirish. |
| `wait "$GOOD_PID"` | Bash | Ikkala mijoz ham so‘rovlarini to‘liq yakunlashini kutish. |
| `grep 'REJECTED' /opt/a12/access.jsonl \| tail -n 6` | grep + tail | Kuzatuv rejimida `REJECTED_DRY_RUN` yozuvlari tushganini tekshirish. |

### 6-Bosqich: Faol Cheklovni Yoqish va "Keyin" (After) Bosqichi
| CLI Buyruq | Qo‘llanilgan Tool | Bajarilish Maqsadi |
| :--- | :--- | :--- |
| `cp /opt/a12/nginx.conf /opt/a12/nginx-before.conf` | Coreutils (`cp`) | Kuzatuv konfiguratsiyasining zaxira nusxasini olish. |
| `sed -i 's/limit_req_dry_run on;/limit_req_dry_run off;/' /opt/a12/nginx.conf` | SED (Stream Editor) | Konfiguratsiya faylidagi qatorni avtomatik ravishda faol rejimga almashtirish. |
| `nginx -t -c ... && nginx -s reload -c ...` | Nginx | Sintaksisni tekshirib, Nginxni to‘xtatmasdan yangi sozlamani qayta yuklash (Hot Reload). |
| *(Xuddi shu parallel sinov buyruqlari `after` tegi bilan qayta bajarildi)* | timeout + netns + Python | Faol cheklov rejimida ikkala mijoz natijalarini CSV fayllarga yozib olish. |
| `grep 'REJECTED' /opt/a12/access.jsonl \| tail -n 8` | grep + tail | Faol rejimda haqiqiy `REJECTED` va HTTP `429` statusi qaytganini logdan tekshirish. |

### 7-Bosqich: Natijalarni Hisoblash va Xavfsizlik Tekshiruvlari
| CLI Buyruq | Qo‘llanilgan Tool | Bajarilish Maqsadi |
| :--- | :--- | :--- |
| `python3 - <<'METRICS' ... METRICS` | Python Inline Skript | 4 ta CSV faylni o‘qib, 200, 429 soni, muvaffaqiyat foizi va p95 kechikishini hisoblab chiqarish. |
| `ip netns exec a12-test curl ... http://10.120.10.1:8080/` | curl + netns | Backend 8080-porti tashqi mijoz uchun yopiq ekanligini (Bypass yo‘qligini) isbotlash. |
| `ip netns exec a12-test curl ... -H 'X-Forwarded-For: 192.0.2.55' ...` | curl + netns | Soxta IP sarlavhasi yuborilganda ham Nginx aldanmaganini (`access.jsonl` da) tekshirish. |

---

# 4-BO‘LIM: SELF-STUDY YO‘L XARITASI (ROADMAP): QAYSI DARAJAGA YETGANDA BULARNI O‘RGANISH KERAK?

Ushbu laboratoriyada bajarilgan ishlar IT sohasining **DevOps, DevSecOps, Tizim Administratorligi (SysAdmin) va Tarmoq Xavfsizligi (Network Security)** yo‘nalishlariga bevosita tegishli.

Quyidagi bosqichma-bosqich yo‘l xaritasi sizga bu mavzularni o‘rganish ketma-ketligini ko‘rsatadi:

```
+-----------------------------------------------------------------------------------------+
|                                KASBIY RIVOJLANISH BOSQICHLARI                           |
+-----------------------------------------------------------------------------------------+
|                                                                                         |
|  [ 1. JUNIOR BOSQICH ] -> Linux CLI, OSI modeli, HTTP asoslari, Nginx oddiy veb-server  |
|           |                                                                             |
|           v                                                                             |
|  [ 2. MIDDLE BOSQICH ] -> (HOZIRGI DARAJA!) Linux Namespaces, Nginx Rate Limit,         |
|                           Reverse Proxy, Leaky Bucket, Bash/Python skriptlash           |
|           |                                                                             |
|           v                                                                             |
|  [ 3. SENIOR BOSQICH ] -> BGP Anycast, Cloudflare/Akamai WAF, eBPF/XDP filtrlash,       |
|                           High-Availability, Tarmoq darajasida DDoS tozalash            |
|                                                                                         |
+-----------------------------------------------------------------------------------------+
```

### 1-Bosqich: Junior (Boshlang‘ich) daraja — Fundament
* **Qachon o‘rganiladi:** Universitetning 1–2 kurslarida yoki mustaqil ta’limning dastlabki oylarida.
* **O‘rganiladigan mavzular:**
  - Linux operatsion tizimi buyruqlari (`ls`, `cd`, `grep`, `chmod`, `systemctl`);
  - Kompyuter tarmoqlari asoslari: OSI va TCP/IP modellari, IP manzillash, MAC manzil, Subnetting;
  - HTTP protokoli: GET/POST metodlari, HTTP status kodlari (200, 301, 404, 500);
  - Nginxni oddiy statik HTML veb-sayt uchun sozlash.

### 2-Bosqich: Middle (O‘rta) daraja — Hozirgi laboratoriya darajasi!
* **Qachon o‘rganiladi:** Universitetning 3–4 kurslarida (5-semestr), Junior lavozimidan Middle darajaga o‘tish davrida.
* **Bu bosqichda nimani bilish shart (A12 da o‘rgangallarimiz):**
  - **Linux Network Namespaces (`ip netns`):** Docker va konteynerlash texnologiyalari (Kubernetes) tagida aynan shu mexanizm yotadi! Siz bugun bitta VMda ikkita mijozni ajratish orqali Docker tarmog‘ining ichki ishlash mohiyatini o‘rgandingiz.
  - **Nginx Reverse Proxy:** Backendni yashirish, yuklamani taqsimlash (Load Balancing), xavfsiz headerlarni o‘rnatish.
  - **Rate Limiting algoritmlari:** Leaky Bucket va Token Bucket qanday ishlashi, `burst` va `nodelay` parametrlari nima uchun kerakligi.
  - **L7 DoS hujumlariga qarshi kurash:** Oddiy foydalanuvchini bloklamasdan tajovuzkorni cheklash san’ati.
  - **Sinovlarni avtomatlashtirish:** Python orqali parallel so‘rovlar yuborish va CSV/JSON orqali natijalarni tahlil qilish.

### 3-Bosqich: Senior / Lead (Yuqori) daraja — Katta tizimlar va global arxitektura
* **Qachon o‘rganiladi:** Real ishlab chiqarish loyihalarida 3–5 yillik tajribadan so‘ng.
* **O‘rganiladigan yuqori darajadagi mavzular:**
  - **L3/L4 Volumetrik hujumlarni qaytarish:** BGP Anycast marshrutlash, Cloudflare Magic Transit, AWS Shield kabi global tarmoqlar qanday ishlashi.
  - **eBPF (Extended Berkeley Packet Filter) va XDP (eXpress Data Path):** Linux yadrosining tarmoq drayveri darajasida paketlarni CPUga chiqarmasdanoq milliardlab paket/s tezlikda tashlab yuborish (Drop qilish). Cloudflare aynan shu texnologiyadan foydalanadi.
  - **WAF (Web Application Firewall):** SQL Injection, XSS va murakkab botnet hujumlarini sun’iy intellekt va semantik tahlil orqali to‘xtatish (masalan, ModSecurity, Coraza).
  - **SLO/SLA monitoringi:** Prometheus, Grafana, Distributed Tracing (Jaeger) yordamida global infratuzilmani 24/7 nazorat qilish.

---

# 5-BO‘LIM: SHAXSIY MASLAHATLAR VA AMALIY TAVSIYALAR

Ushbu amaliyotdan kelib chiqib, kelgusida haqiqiy loyihalarda xatoga yo‘l qo‘ymasligingiz uchun **7 ta oltin qoida**:

1. **Hech qachon ishlab turgan (Production) tizimda birdaniga `dry_run off` qilmang!**
   * *Nega?* Chunki siz qoidani qanchalik to‘g‘ri yozdim deb o‘ylamang, real foydalanuvchilar orasida sahifani tez-tez yangilaydigan (F5 bosadigan) yoki bir vaqtda 10 ta rasm yuklaydiganlar bo‘ladi.
   * *Tavsiya:* Yangi qoidani kamida 24–48 soat `dry_run on` rejimida ushlab, `error.log` va `access.jsonl` dagi `REJECTED_DRY_RUN` yozuvlarini tahlil qiling. Faqat qonuniy mijozlar bloklanmasligiga 100% ishonch hosil qilgach, faol rejimga o‘ting.

2. **IP manzilga ko‘r-ko‘rona ishonmang (NAT va CGNAT muammosi):**
   * Bugungi 6.1-hisob-kitobda ko‘rganingizdek, bitta NAT yoki mobil operator (Beeline, Ucell va h.k.) ortida minglab odamlar bitta IP bilan o‘tirgan bo‘lishi mumkin.
   * *Tavsiya:* Rate limitingni faqat IP manzilga emas, iloji boricha **autentifikatsiyadan o‘tgan foydalanuvchi identifikatori (Session ID, JWT token yoki API Key)** bo‘yicha o‘rnating.

3. **Backendni hech qachon tashqi dunyoga ochiq qoldirmang (Origin Shielding):**
   * Ko‘p uchraydigan qo‘pol xato — serverda Nginx va WAF o‘rnatiladi-da, lekin dasturchi backendni `0.0.0.0:8000` ga ochib qo‘yadi. Tajovuzkor domen orqali emas, to‘g‘ridan-to‘g‘ri serverning IPsi va 8000 portiga hujum qilib, barcha WAF va Nginx himoyalarini bir zumda chetlab o‘tadi (Bypass).
   * *Tavsiya:* Backend faqat `127.0.0.1` (loopback) yoki alohida xususiy ichki tarmoqda (Private VPC) tinglashi shart!

4. **Kechikish (p95) statistikasida 429 xatolarining "tuzog‘iga" tushmang:**
   * Laboratoriyada ko‘rganingizdek, Nginx 429 xatosini 1 millisekundda qaytaradi. Agar tajovuzkor 10,000 ta so‘rov yuborsa va hammasi 1 ms da 429 olsa, tizimning o‘rtacha javob vaqti "yaxshilangan"dek ko‘rinadi. Bu soxta ko‘rsatkich.
   * *Tavsiya:* Har doim faqat muvaffaqiyatli (`status == 200`) so‘rovlar kechikishini alohida metrika sifatida kuzating!

5. **L7 himoyasi bilan L3/L4 to‘fonini to‘xtatib bo‘lmasligini unutmang:**
   * Nginx qanchalik zo‘r sozlanmasin, agar provayderdan sizga kelayotgan $100\text{ Mbit/s}$ lik jismoniy kabelga tajovuzkor $1\text{ Gbit/s}$ lik to‘fon yuborsa, Nginxga paketlar yetib ham kelmaydi.
   * *Tavsiya:* Hajmli hujumlarga qarshi faqat tarmoq provayderi (ISP) darajasida yoki Cloudflare/AWS CloudFront kabi global CDNlar orqali himoyalaniladi.

6. **Mijoz yuborgan `X-Forwarded-For` ga hech qachon ishonchsiz ruxsat bermang:**
   * Tajovuzkor o‘z so‘roviga `X-Forwarded-For: 8.8.8.8` deb xohlagan IP manzilini yozib yuborishi mumkin.
   * *Tavsiya:* Nginxda faqat oldinda turgan aniq ishonchli yuklama taqsimlagich (Load Balancer) IP manzillari uchungina `set_real_ip_from` direktivasini ishlating.

7. **Virtualizatsiya va Linux Namespaces bo‘yicha amaliyotingizni davom ettiring:**
   * Bugun siz Windowsda emas, haqiqiy **Ubuntu Linux** ichida **VMware** va **Linux Network Namespaces** vositasida professional laboratoriya qildingiz. Ushbu yondashuv sizga Docker konteynerlarining tarmoq arxitekturasini tushunishda ulkan poydevor beradi.

---

# 6-BO‘LIM: YAKUNIY XULOSA

A12 amaliy mashg‘uloti zamonaviy veb-xizmatlarning barqarorligi va mavjudligini ta’minlash bo‘yicha fundamental tushunchalarni amalda sinash imkonini berdi.

**Xulosa qilib aytganda:**
1. **Nazariy bilim amaliyot bilan mustahkamlandi:** DoS va DDoS farqi, L3/L4 va L7 hujumlarining nishonlari ajratildi.
2. **Nginx imkoniyatlari to‘liq ochib berildi:** Rate limiting shunchaki barcha narsani bloklash emas, balki qonuniy oqimga zarar yetkazmasdan tizim resurslarini himoyalash san’ati ekanligi dalillar (CSV va JSON loglar) bilan isbotlandi.
3. **Qatlamli himoya (Defense-in-Depth) zarurligi anglandi:** Hech qanday bitta vosita (faqat Nginx yoki faqat provayder) to‘liq xavfsizlikni bera olmaydi. Xavfsizlik — bu Provayder, CDN, WAF, Reverse Proxy, Operatsion tizim va Ilova darajalarining o‘zaro hamjihatlikda ishlashidir.

Ushbu amaliyotda olingan ko‘nikmalar sizning kelgusidagi DevOps, tizim administratori yoki kiberxavfsizlik mutaxassisi sifatidagi kasbiy faoliyatingizda eng kuchli va mustahkam asoslardan biri bo‘lib xizmat qiladi!
