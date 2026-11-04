# TARMOQ VA VEB-XIZMATLAR XAVFSIZLIGI | A13 AMALIYOT TAHLILI
## "Proxy va WAF Vositalarini Sozlash" Mavzusi Bo‘yicha Xulosa, Tizimli Tahlil va Rivojlanish Yo‘l Xaritasi

Ushbu hujjat **`description.md`** va **`TX_A13_1_variant_batafsil.pdf`** materiallarida bajarilgan barcha amaliy ishlarning tub mohiyatini, ishlatilgan vositalar zanjirini, buyruqlar maqsadini hamda kelajakda professional mutaxassis bo‘lib yetishishdagi o‘rnini umumlashtirib beruvchi yakuniy hisobot hisoblanadi.

---

## 1. VAZIFANI BAJARISHDAN ASOSIY MAQSAD NIMA EDI?

Ushbu amaliy mashg‘ulot shunchaki tayyor dastur o‘rnatish emas, balki korporativ ishlab chiqarish (**Production**) muhitlarida qo‘llaniladigan **ko‘p qatlamli himoya (Defense-in-Depth)** arxitekturasini noldan qurish va isbotlashga qaratilgan.

### Asosiy maqsadlar:
1. **Origin (asl) serverni yashirish va izolyatsiyalash:**
   - Ilovaning asosiy biznes mantig‘i bajariladigan Python backend serverini faqat operatsion tizimning ichki xotirasida — `127.0.0.1:8080` (**Loopback**) manzilida ishlatish.
   - Tashqi tarmoq mijozlari backend portiga to‘g‘ridan-to‘g‘ri ulanishini to‘liq imkonsiz qilish (**WAF Bypass** hujumlarining oldini olish).
2. **Reverse Proxy (Teskari Proksi) orqali xavfsiz shlyuz yaratish:**
   - Backend oldiga **Apache 2.4** teskari proksisini joylashtirib, tashqi trafikni yagona nuqtadan qabul qilish.
   - Ichki infratuzilma haqidagi ma’lumotlar tashqariga sizib chiqmasligi uchun HTTP javob sarlavhalarini (`ProxyPassReverse` yordamida `Location` sarlavhasini) qayta yozish va soxtalashtirilishi mumkin bo‘lgan sarlavhalarni tozalash (`X-Forwarded-For`).
3. **OSI 7-qatlamida (L7) WAF himoyasini o‘rnatish:**
   - Veb-ilovalarni tahlil qiluvchi **ModSecurity 2.x** dvigatelini o‘rnatish va unga dunyo standarti hisoblangan **OWASP Core Rule Set (CRS)** qoidalarini ulash.
   - WAFning ikki muhim rejimini amalda farqlash:
     - **Kuzatuv rejimi (`DetectionOnly`):** Tizim faoliyatiga xalaqit bermasdan, hujumlarni aniqlab jurnalga yozish;
     - **Faol bloklash rejimi (`On`):** Tahdid aniqlangan zahoti so‘rovni backendga yetkazmasdan HTTP 403 bilan rad etish.
4. **False Positive (Soxta Ijobiy) muammosini professional darajada hal qilish (WAF Tuning):**
   - Veb-xavfsizlikdagi eng katta amaliy muammo — qonuniy foydalanuvchining zararsiz so‘rovi ("JavaScript darsi") qoida tomonidan xato tarzda "hujum" deb baholanib, bloklanishi.
   - Butun WAFni o‘chirib qo‘ymasdan, faqatgina zarur endpoint (`/submit`), usul (`POST`), qoida ID (`1001301`) va parametr (`ARGS:comment`) uchun **tor runtime istisno (fine-grained rule exclusion)** yaratish va tizimning qolgan himoyasini (XSS ga qarshi CRS himoyasini) 100% saqlab qolish.
5. **Korrelyatsion tahlil va dalillarni bog‘lash:**
   - Har bir HTTP tranzaksiyasiga berilgan noyob **`UNIQUE_ID`** orqali mijoz so‘rovi, Apache access log, ModSecurity error log, to‘liq audit log va backend jurnalini o‘zaro zanjir qilib bog‘lash.

---

## 2. VAZIFANI BAJARISHDA QANDAY TASHQI KUTUBXONA VA TOOL-LARDAN FOYDALANILDI?

Mashg‘ulot davomida bir nechta mustaqil dasturiy vositalar va modullar o‘zaro integratsiya qilindi:

| Vosita / Kutubxona | Kategoriyasi | Ushbu Laboratoriyadagi Vazifasi va Roli |
| :--- | :--- | :--- |
| **Apache 2.4 (`apache2`)** | Web Server / Gateway | Asosiy teskari proksi platformasi. Mijoz so‘rovlarini qabul qilib, WAF orqali filtrlaydi va backendga uzatadi. |
| **`mod_proxy` & `mod_proxy_http`** | Apache Dinamik Moduli | Proksilash mexanizmini va HTTP protokoli orqali ichki backend bilan aloqani ta’minlaydi. |
| **`mod_headers`** | Apache Moduli | HTTP sarlavhalarini manipulyatsiya qiladi (soxta IP sarlavhalarini o‘chiradi, `X-A13-Gateway` belgisini qo‘shadi). |
| **`mod_unique_id`** | Apache Moduli | Har bir HTTP so‘rovi uchun takrorlanmas 24 belgili `UNIQUE_ID` kodini generatsiya qiladi. |
| **`libapache2-mod-security2`** | Web Application Firewall | ModSecurity v2.x dvigateli. HTTP so‘rov tanasi va sarlavhalarini chuqur tekshiradi (DPI). |
| **`modsecurity-crs`** | Xavfsizlik Qoidalari To‘plami | OWASP Core Rule Set (CRS). XSS, SQLi, RCE kabi hujumlarni aniqlash va anomaliya ballarini yig‘ish bazasi. |
| **Python 3 (`http.server`, `urllib`, `json`)** | Dasturlash Tili / Standart Kutubxona | O‘quv veb-servisi (REST API). Tashqi kutubxonalarsiz, sof loopbackda ishlovchi backend yaratish uchun ishlatildi. |
| **Systemd** | Linux Init / Service Manager | Python backendni xavfsiz tizim xizmati sifatida ishga tushirish, qayta yuklash va izolyatsiyalash vositasi. |
| **`curl` (Linux CLI)** | HTTP Mijoz Vosita | So‘rovlarni turli sarlavhalar, parametrlar va test yuklamalari bilan yuborish vositasi. |
| **`nc` (Netcat)** | Tarmoq Diagnostika Vosita | Backend porti (8080) tashqi tarmoqdan yopiq ekanligini isbotlash uchun port skaneri sifatida ishlatildi. |
| **`ss` (Socket Statistics)** | Tarmoq Tahlili | Ochiq portlar va tinglovchi interfeyslarni tekshirish vositasi. |
| **VMware Virtual Network Editor** | Gipervizor Tarmoq Vosita | Tashqi Wi-Fi dan ajratilgan, xavfsiz Host-Only (`10.130.10.0/24`) tarmog‘ini barpo etish vositasi. |

---

## 3. QADAM-BAQADAM CLI BUYRUQLAR VA ULARNING ISHLATILISH MAQSADI

Laboratoriya davomida berilgan barcha buyruqlar mantiqiy ketma-ketlikda quyidagi aniq maqsadlar uchun bajarildi:

### 1-Bosqich: Paketlarni o‘rnatish va tarmoqni sozlash
* **`sudo apt update`**
  - *Maqsadi:* Ubuntu repozitoriyalaridagi eng so‘nggi paketlar indeksini yangilash.
* **`sudo apt install -y apache2 libapache2-mod-security2 modsecurity-crs python3 curl nano`**
  - *Maqsadi:* Laboratoriya uchun zarur bo‘lgan veb-server, WAF moduli, OWASP qoidalari va yordamchi utilitalarni o‘rnatish.
* **`ip link set ens33 up && ip addr replace 10.130.10.10/24 dev ens33`**
  - *Maqsadi:* VMware Host-Only tarmog‘iga ulangan virtual interfeysga statik IP belgilash.
* **`dpkg-query -W ...` va `dpkg -L modsecurity-crs`**
  - *Maqsadi:* O‘rnatilgan paketlar versiyasini va CRS qoidalarining diskdagi aniq katalog yo‘llarini aniqlash.

### 2-Bosqich: Python Backendni tayyorlash va izolyatsiyalash
* **`cat > /opt/a13/backend.py <<'APP' ... APP`**
  - *Maqsadi:* Faqat `127.0.0.1:8080` manzilida tinglovchi, JSON ma’lumot qaytaruvchi va 302 redirekt beruvchi o‘quv ilovasini yaratish.
* **`python3 -m py_compile /opt/a13/backend.py`**
  - *Maqsadi:* Python skriptida sintaktik xatolar yo‘qligini tekshirish.
* **`cat > /etc/systemd/system/a13-backend.service <<'UNIT' ... UNIT`**
  - *Maqsadi:* Ilovani `www-data` imtiyozsiz foydalanuvchisi va `NoNewPrivileges=true`, `PrivateTmp=true` xavfsizlik parametrlari bilan tizim xizmatiga aylantirish.
* **`systemctl daemon-reload && systemctl enable --now a13-backend`**
  - *Maqsadi:* Xizmatni systemd ro‘yxatiga qo‘shib, darhol fonda ishga tushirish.
* **`ss -lntp 'sport = :8080'`**
  - *Maqsadi:* Backend faqat loopbackda (`127.0.0.1:8080`) tinglayotganini, tashqi interfeyslarga ochiq emasligini tekshirish.

### 3-Bosqich: Apache Reverse Proxy-ni ishga tushirish
* **`cp -a /etc/apache2 /opt/a13/apache2-before`**
  - *Maqsadi:* Konfiguratsiya buzilsa, dastlabki holatga qaytish uchun to‘liq zaxira (backup) olish.
* **`a2enmod proxy proxy_http headers unique_id security2`**
  - *Maqsadi:* Apache ichida proksi, sarlavhalar, noyob ID va WAF modullarini faollashtirish.
* **`printf 'Listen 10.130.10.10:80\n' > /etc/apache2/ports.conf`**
  - *Maqsadi:* Apache standart hamma interfeyslarda emas, faqat lab tarmog‘ida tinglashini ta’minlash.
* **`cat > /etc/apache2/sites-available/a13.conf ... a2ensite a13`**
  - *Maqsadi:* VirtualHost yaratish: `ProxyPass` orqali trafikni backendga yo‘naltirish, `ProxyPassReverse` bilan sarlavhalarni to‘g‘rilash, `LogFormat` orqali `UNIQUE_ID` ni jurnalga kiritish.
* **`apache2ctl configtest && systemctl restart apache2`**
  - *Maqsadi:* Sintaksisni tekshirish ("Syntax OK") va serverni xatosiz qayta ishga tushirish.

### 4-Bosqich: ModSecurity va CRS qoidalar zanjirini ulash
* **`cp ...modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf`**
  - *Maqsadi:* Tavsiya etilgan xavfsizlik shablonini asosiy sozlama sifatida nusxalash.
* **`sed -i 's/^SecRuleEngine .*/SecRuleEngine DetectionOnly/' ...`**
  - *Maqsadi:* WAFni birinchi bosqichda faqat kuzatuv (bloklamaydigan) rejimiga sozlash.
* **`cat > /etc/apache2/mods-available/security2.conf ...`**
  - *Maqsadi:* Qoidalar yuklanish tartibini qat’iy belgilash: `modsecurity.conf` -> `a13-before.conf` (istisnolar) -> `crs-setup.conf` -> CRS qoidalari -> `a13-local.conf`.
* **`apache2ctl -t -D DUMP_INCLUDES`**
  - *Maqsadi:* Qoidalar fayllari xotiraga aynan qaysi tartibda yuklanganini daraxt ko‘rinishida chiqarib, tekshirish.

### 5-Bosqich: Mahalliy qoidalar va Sinovlar (T1–T12)
* **`cat > /etc/modsecurity/a13-local.conf ...`**
  - *Maqsadi:* Sinov uchun ikkita qoida qo‘shish: `1001300` (sarlavhadagi marker) va `1001301` (formadagi "script" so‘zi).
* **`curl ... -H "X-A13-Case: T1" /health` (T1, T2, T3)**
  - *Maqsadi:* Kuzatuv rejimida oddiy so‘rov, marker va XSS kiritmasi yuborilganda barchasi 200 qaytarishini, ammo error logda qoidalar qayd etilganini isbotlash.
* **`sed -i 's/^SecRuleEngine .*/SecRuleEngine On/' ...`**
  - *Maqsadi:* WAFni to‘liq faol bloklash rejimiga o‘tkazish.
* **`curl ... (T4, T5, T6)`**
  - *Maqsadi:* Faol rejimda T4 (normal) 200 olishini, T5 (marker) 403 olishini va T6 (CRS XSS) 403 bilan bloklanishini isbotlash.
* **`curl ... --data-urlencode "comment=JavaScript darsi" (T7)`**
  - *Maqsadi:* Qonuniy matn Rule 1001301 tomonidan xato bloklanganini (**False Positive**) keltirib chiqarish.
* **`cat > /etc/modsecurity/a13-before.conf ... (Rule 1001302 istisnosi)`**
  - *Maqsadi:* Faqat `/submit` va `POST` uchun `1001301` qoidasidan `ARGS:comment` nishonini olib tashlovchi tor istisno kiritish.
* **`curl ... (T8, T9, T10)`**
  - *Maqsadi:* T8 da qonuniy izoh o‘tganini (200), T9 da boshqa endpointda qoida saqlanganini (403), T10 da esa XSS hujumi CRS tomonidan baribir bloklanganini (403) isbotlash.
* **`nc -zv 10.130.10.10 8080 (T11)`**
  - *Maqsadi:* Backend porti tashqi mijoz uchun yopiq ekanligini (`Connection refused`) ko‘rsatish.
* **`curl -i $U/redirect (T12)`**
  - *Maqsadi:* `Location` sarlavhasida ichki `127.0.0.1:8080` fosh bo‘lmasdan, proksi manziliga o‘zgarganini tasdiqlash.

---

## 4. SELF-STUDY QILGANDA BU ISHLARNI QAYSI DARAJAGA YETGANDA O‘RGANISH KERAK?

Ushbu laboratoriyada bajarilgan bilimlar IT mutaxassisining karera yo‘lida qaysi bosqichda va qaysi sohalarda talab etiladi?

```
+-----------------------------------------------------------------------------------+
| 1. JUNIOR BOSQICHI (Foundation):                                                  |
|    - Linux CLI, Bash asoslari, fayl tizimi huquqlari.                             |
|    - TCP/IP modeli, HTTP so'rov/javob tuzilishi, portlar.                         |
|    - Web-serverlar (Nginx/Apache)da oddiy sayt va proksi ko'tarish.               |
+------------------------------------------+----------------------------------------+
                                           |
                                           v
+-----------------------------------------------------------------------------------+
| 2. MIDDLE / PRE-SENIOR BOSQICHI (Siz hozir turgan daraja):                        |
|    - Reverse Proxy orqali arxitektura qurish va originni yashirish.              |
|    - ModSecurity WAF va OWASP CRS qoidalarini joriy qilish.                       |
|    - DetectionOnly vs On rejimlarini boshqarish.                                  |
|    - False Positive muammolarini hal qilish (WAF Tuning / Exclusions).            |
|    - Audit va access loglarni UNIQUE_ID orqali korrelyatsiya qilish.              |
+------------------------------------------+----------------------------------------+
                                           |
                                           v
+-----------------------------------------------------------------------------------+
| 3. SENIOR & PROFESSIONAL MUTAXASSISLIK YO'NALISHLARI:                             |
|    - AppSec Engineer (Application Security): Kod zaifliklariga Virtual Patching   |
|      qilish, xakerlik hujumlarini WAF qoidalariga aylantirish.                    |
|    - DevSecOps Engineer: CI/CD quvurida WAFni avtomatlashtirish, Ingress          |
|      Controller (Kubernetes) darajasida ModSecurity/Coraza integratsiyasi.        |
|    - SOC Analyst / Blue Team: WAF loglarini SIEM tizimlariga yig'ish va tahlil.  |
+-----------------------------------------------------------------------------------+
```

### Xulosa:
Agar siz o‘zingiz mustaqil o‘rganayotgan bo‘lsangiz:
- Ushbu mavzuga **Linux va Tarmoq asoslarini (CCNA / Linux Essentials darajasida) to‘liq o‘zlashtirib bo‘lgach** kirishish kerak.
- Bu mavzu bevosita **Junior darajasidan Middle darajasiga sakrash nuqtasi** hisoblanadi. Chunki oddiy administrator shunchaki saytni ishga tushiradi, ammo **Middle muhandis** saytni qatlamli himoya qiladi, WAF o‘rnatadi va mijoz trafigi buzilmasligi uchun qoidalarni nozik sozlay oladi (**Tuning**).

---

## 5. SIZ UCHUN SHAXSIY TAVSIYALAR VA XULOSA

### 💡 5 ta Oltin Maslahat:

1. **"WAF — bu dasturchining xatosini kechiruvchi afsungar emas!"**
   - Hech qachon WAF bor deya dasturiy kodni zaif yozmaslik kerak. Agar dasturda SQL Injection yoki XSS bo‘lsa, WAF uni vaqtincha to‘sib turishi mumkin, ammo malakali xaker WAFni aylanib o‘tish yo‘lini (*WAF Evasion*) baribir topadi.
   - WAFning asosiy vazifasi — dasturchi xatosini tuzatguncha tizimga **vaqt yutib berish (Virtual Patching)** va ommaviy skanerlar/botlarni to‘xtatishdir.
2. **"Hech qachon ishlab turgan tizimda WAFni birdaniga `On` qilib yoqmang!"**
   - Yangi loyihaga WAF o‘rnatganda, uni kamida 1–2 hafta faqat **`DetectionOnly`** rejimida ishlatish shart.
   - Bu davrda barcha loglarni tahlil qilib, qaysi qonuniy so‘rovlar (masalan, to‘lov tizimlari, foydalanuvchi izohlari) qoidaga tushayotganini aniqlash va ular uchun oldindan istisno (**Exclusion**) yozish kerak. Aks holda sayt mijozlari uchun ishlamay qoladi va biznes zarar ko‘radi.
3. **"Qoidani butunlay o‘chirish — eng yomon ma’murning ishi!"**
   - Laboratoriyada ko‘rganingizdek, `"JavaScript darsi"` bloklanganida eng oson yo‘l `SecRuleEngine Off` qilish edi. Ammo biz bunday qilmadik. Har doim faqat muammo chiqqan parametr va endpoint uchun **tor doirada** istisno yozish madaniyatini shakllantiring.
4. **"Origin serverni har doim dunyodan yashiring!"**
   - Qanchalik zo‘r WAF qo‘ymang, agar sizning backend ilovangiz `0.0.0.0` da tinglab tursa va xaker uning IP manzilini bilib olsa, u to‘g‘ridan-to‘g‘ri portga ulanib, butun WAFni aylanib o‘tadi. Shuning uchun faqat `127.0.0.1` (yoki yopiq ichki VPN/VPC tarmog‘i) orqali ulanishga ruxsat bering.
5. **Amaliyotni virtualizatsiyada davom ettiring:**
   - O‘zingizning Ubuntu + VMware muhitingizda alohida kichik lablar qiling. Docker konteynerlarida **OWASP ModSecurity Core Rule Set (CRS) Docker** obrazlarini ko‘tarib ko‘ring. Bu sizga DevSecOps bo‘yicha ulkan amaliy tajriba beradi.

---

### 🎯 Yakuniy Xulosa
Siz A13 laboratoriya mashg‘ulotida zamonaviy kiberxavfsizlikning fundamental tamoyillarini to‘liq qamrab oldingiz:
- Tarmoq izolyatsiyasi (Loopback va Host-Only);
- L7 darajasidagi so‘rovlar filtrlash (ModSecurity);
- Xalqaro xavfsizlik standartlari (OWASP CRS);
- Loglar orqali voqealarni tekshirish va isbotlash (UNIQUE_ID korrelyatsiyasi).

Ushbu amaliyot sizni nafaqat laboratoriya topshirig‘ini a’lo bahoga topshirishingizga, balki kelajakda kuchli **Tarmoq Xavfsizligi Muhandisi** yoki **DevSecOps Mutaxassisi** bo‘lib shakllanishingizga mustahkam poydevor bo‘lib xizmat qiladi!
