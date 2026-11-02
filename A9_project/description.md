# A9: WIRESHARK VA NETFLOW YORDAMIDA TARMOQ MONITORINGI
## 1-Variant: Paketlar va Oqimlarni O‘zaro Taqqoslash
### To‘liq Amaliy va Metodik Qo‘llanma (Ubuntu Host + VMware Muhiti Uchun Moslashtirilgan)

---

## MUNDARIJA
1. [Mavzuning Maqsadi va Dolzarbligi](#1-mavzuning-maqsadi-va-dolzarbligi)
2. [Nazariy Tushunchalar va Texnologiyalar Tahlili](#2-nazariy-tushunchalar-va-texnologiyalar-tahlili)
   - 2.1. Paket Tahlili (PCAP / Wireshark) va Oqim Tahlili (NetFlow / IPFIX)
   - 2.2. NetFlow v9 Arxitekturasi: Exporter, Collector va Shablonlar (Templates)
   - 2.3. Tarmoq Sarlavhalari: Tashqi (Outer) IP vs Ichki (Inner) Flow Ma'lumotlari
   - 2.4. Oqimlarning Bo‘linishi (Flow Expiry: Active Timeout va Inactive Timeout)
   - 2.5. Wireshark Analitik Mexanizmlari (Filters, Conversations, I/O Graphs, Streams)
3. [Laboratoriya Muhiti va VMware Sozlamalari (Ubuntu Host va Shaxsiy Wi-Fi)](#3-laboratoriya-muhiti-va-vmware-sozlamalari)
   - 3.1. Shaxsiy Wi-Fi va LAN Segment Izolyatsiyasi
   - 3.2. VMware Workstation (Ubuntu) da Virtual Tarmoqlarni Sozlash
   - 3.3. Dasturiy Ta'minotlarni O'rnatish va Wireshark (Ubuntu Host)
   - 3.4. Topologiya va IP Manzillash Jadvali
4. [Mashqni Bajarish Bo‘yicha Qadamma-Qadam Qo‘llanma va CLI Buyruqlar Tahlili](#4-mashqni-bajarish-boyicha-qadamma-qadam-qollanma)
   - 4.1. 1-Bosqich: Tarmoq Interfeyslari va Statik Marshrutlarni Sozlash
   - 4.2. 2-Bosqich: Tarmoq Bog'lanishi va Vaqtni Tekshirish
   - 4.3. 3-Bosqich: HTTP Xizmatini Ishga Tushirish (SRV-MON)
   - 4.4. 4-Bosqich: NetFlow Kollektori (nfcapd) va Paket Yozuvini (tcpdump) Boshlash
   - 4.5. 5-Bosqich: NetFlow Eksportchisini (softflowd) Sozlash va Boshlash (RTR-1)
   - 4.6. 6-Bosqich: Nazoratli Trafik Hosil Qilish (CLIENT)
   - 4.7. 7-Bosqich: Oqimlarni Yakunlash, Saqlash va Jarayonlarni To‘xtatish
   - 4.8. 8-Bosqich: Dalillarni Ubuntu Hostga Ko‘chirish
   - 4.9. 9-Bosqich: Wireshark Yordamida Paketlarni Tahlil Qilish
   - 4.10. 10-Bosqich: nfdump Yordamida Oqimlar Hisobotini Olish (SRV-MON)
   - 4.11. 11-Bosqich: Bitta Ulanishni Wireshark va NetFlowda O'zaro Solishtirish
   - 4.12. 12-Bosqich: Xavfsizlik Hodisasini Tahlil Qilish va Monitoring Qoidasini Ishlab Chiqish
5. [Nosozliklarni Bartaraf Etish (Troubleshooting) va Og‘zaki Himoya Savollari](#5-nosozliklarni-bartaraf-etish-va-ogzaki-himoya)
   - 5.1. Amaliyotda Uchraydigan Xatoliklar va Ularning Yechimlari
   - 5.2. Og'zaki Himoya Savollari va Ilmiy-Amaliy Javoblar
6. [Foydalanilgan va Tavsiya Etiladigan Manbalar](#6-foydalanilgan-va-tavsiya-etiladigan-manbalar)

---

## 1. MAVZUNING MAQSADI VA DOLZARBLIGI

### 1.1. Asosiy Maqsad
Ushbu laboratoriya mashg‘ulotining asosiy maqsadi — tarmoq xavfsizligi va monitoringida qo‘llaniladigan ikki xil fundamental yondashuvni: **paketlar darajasidagi to‘liq tahlil (Full Packet Capture — PCAP)** va **oqimlar darajasidagi yig‘ma statistik tahlilni (Flow-based Analysis — NetFlow v9)** ajratilgan virtual laboratoriya tarmog‘ida amaliy o‘rganish, ularning ishlash mexanizmlarini taqqoslash hamda tarmoq anomaliyalarini aniqlash ko‘nikmalarini shakllantirishdan iborat.

### 1.2. Kutiladigan Natijalar va O‘zlashtiriladigan Ko‘nikmalar
Ushbu mashg‘ulotni to‘liq bajargan talaba:
1. **Tarmoq rollarini ajratish:** Trafik generatori (Mijoz / CLIENT), tranzit marshrutizator hamda telemetriya sensori (Eksportchi / RTR-1) va xizmat ko‘rsatuvchi server hamda markaziy xotira (Kollektor / SRV-MON) vazifalarini chuqur tushunadi.
2. **Paket tahlili (Wireshark/tcpdump):** Paketlarni to‘liq ushlash (capture), BPF (Berkeley Packet Filter) filtrlari, Display filtrlar, `TCP Conversations`, `I/O Graphs` va `Follow TCP Stream` vositalari bilan ishlashni o‘rganadi.
3. **Oqim telemetriyasi (NetFlow v9):** `softflowd` eksportchisi, `nfcapd` kollektori va `nfdump` analizatori yordamida oqim ma'lumotlarini to‘plash, filtrlash, agregatsiya qilish va statistik saralashni (Top Talkers / Top Ports) bajaradi.
4. **Ichki va tashqi sarlavhalarni farqlash:** NetFlow eksport paketining tashqi transport sarlavhasi (Router IP $\rightarrow$ Kollektor IP, UDP 2055) bilan uning ichki payload qismidagi kuzatilgan oqim atributlari (Mijoz IP $\rightarrow$ Server IP, TCP 8000) o‘rtasidagi farqni aniqlay oladi.
5. **Gibrid monitoring tahlili:** Bir xil tarmoq hodisasi (masalan, 1 MiB hajmdagi fayl yuklanishi yoki portlarni skanerlash) paketlar darajasida qanday, oqimlar darajasida qanday aks etishini ko‘rib chiqadi, paket va bayt hisoblagichlaridagi farqlar sababini (L2/L3/L4 sarlavhalar, retransmissiyalar, timeoutlar) ilmiy asoslaydi.
6. **Xavfsizlik monitoringi qoidasi:** Anomaliya yoki hujum harakatlarini (masalan, qisqa vaqt ichida ko‘p miqdorda ma'lumot sizdirilishi yoki yopiq portlarga murojaatlar) aniqlash uchun SIEM/NIDS qoidasi loyihasini ishlab chiqadi.

---

## 2. NAZARIY TUSHUNCHALAR VA TEXNOLOGIYALAR TAHLILI

### 2.1. Paket Tahlili (PCAP / Wireshark) va Oqim Tahlili (NetFlow / IPFIX)
Tarmoq ma'murlari va xavfsizlik mutaxassislari (SOC tahlilchilari) tarmoqni kuzatishda ikki xil texnologiyadan foydalanadilar:

| Ko‘rsatkich | Paket Tahlili (PCAP / Wireshark / tcpdump) | Oqim Tahlili (NetFlow / IPFIX / nfdump) |
| :--- | :--- | :--- |
| **Tahlil darajasi** | Har bir paketning barcha sarlavhalari va payload (mazmun) qismi to‘liq yoziladi (Deep Packet Inspection — DPI). | Faqat paketlar metadata yig‘indisi (sarlavhalar statistikasi) yoziladi. Mazmun (payload) saqlanmaydi. |
| **Xotira sarfi** | Juda yuqori (Gigabit tarmoqlarda daqiqasiga gigabaytlab disk joyi talab qiladi). | Juda past (odatda xom paket hajmining 0.1% — 1% qismini tashkil etadi). |
| **Qo‘llanilish sohasi** | Aniq hodisalarni (forensics), eksploytlarni, xatoliklarni chuqur tahlil qilish, shifrlanmagan parollar va sessiyalarni tekshirish. | Katta hajmli magistral tarmoqlarni umumiy kuzatish, trafik trendlari, DDoS hujumlarini aniqlash, hisob-kitob (billing). |
| **Shifrlangan trafik** | TLS/SSL payload ko‘rinmaydi, faqat handshake (SNI, sertifikat) va paket o‘lchamlari ko‘rinadi. | Trafik shifrlangan bo‘lsa ham IP, port, vaqt, paketlar va baytlar soni to‘liq ko‘rinadi. |

### 2.2. NetFlow v9 Arxitekturasi: Exporter, Collector va Shablonlar (Templates)
NetFlow arxitekturasi uchta asosiy komponentdan tashkil topgan:
1. **Sensor / Exporter (Eksportchi):** Tarmoq qurilmasi (marshrutizator, kommutator yoki Linux serverdagi `softflowd`). U interfeysdan o‘tayotgan paketlarni kuzatadi, xotirasidagi oqimlar jadvalida (flow cache) ularni 5-belgi (5-tuple) bo‘yicha guruhlaydi va hisoblagichlarni yangilaydi.
2. **Collector (Kollektor):** Eksportchilardan kelayotgan UDP paketlarni (odatda 2055, 9995 yoki 9996 portlarida) qabul qilib, ularni diskka binar formatda saqlaydigan server xizmati (`nfcapd`).
3. **Analyzer (Tahlilchi):** Saqlangan oqim fayllarini o‘qib, hisobotlar, grafiklar va xavfsizlik tahlillarini yaratuvchi dastur (`nfdump`).

```
[ CLIENT ] --(Traffik)--> [ RTR-1 (softflowd Exporter) ] --(Traffik)--> [ SRV-MON (Server) ]
                                    |
                         (NetFlow v9 UDP/2055)
                                    v
                         [ SRV-MON (nfcapd Collector) ]
                                    |
                                    v
                         [ nfdump / Wireshark ]
```

#### NetFlow v9 Shablonlari (Templates) Mexanizmi
NetFlow v5 qat'iy belgilangan formatga ega bo‘lgan (faqat IPv4 va cheklangan maydonlar). NetFlow v9 (va uning IETF standarti bo‘lgan IPFIX — RFC 7011) esa **dinamik va kengaytiriluvchi shablonli arxitekturaga** asoslangan:
- Eksportchi kollektorga ma'lumot jo‘natishdan oldin **Template FlowSet** (Shablonlar to‘plami) yuboradi. Ushbu shablonda: *"1-maydon — Manba IP (4 bayt), 2-maydon — Maqsad IP (4 bayt), 3-maydon — Paketlar soni (4 bayt)..."* kabi ma'lumot tuzilishi ko‘rsatiladi.
- So‘ngra **Data FlowSet** (Ma'lumotlar to‘plami) yuboriladi. U faqat qiymatlarni o‘z ichiga oladi.
> [!IMPORTANT]
> **Kritik Qoida:** Agar kollektor yoki Wireshark eksportchidan shablon (Template) paketini ushlay olmasa, u Data paketlaridagi binar baytlarni qaysi maydonga tegishli ekanligini ajrata olmaydi va paketlarni ocholmaydi. Shu sababli eksportchini to‘xtatishdan oldin `send-template` buyrug‘i beriladi!

### 2.3. Tarmoq Sarlavhalari: Tashqi (Outer) IP vs Ichki (Inner) Flow Ma'lumotlari
Amaliyotda yangi o‘rganuvchilar ko‘p adashadigan muhim tushuncha:
- **Tashqi transport sarlavhasi (Outer IP/UDP):** Bu NetFlow telemetriyasini yetkazib beruvchi paketning o‘zi.
  - Manba IP: `10.90.20.1` (RTR-1 routerining eksport interfeysi)
  - Maqsad IP: `10.90.20.20` (SRV-MON serverining IP manzili)
  - Protokol: UDP, Port: `2055`
- **Ichki oqim ma'lumotlari (Inner Payload / Flow Record):** NetFlow paketi ichidagi ma'lumotlar yozuvi. Bu router interfeysidan o‘tgan real foydalanuvchi trafigidir.
  - Manba IP: `10.90.10.10` (Mijoz kompyuteri)
  - Maqsad IP: `10.90.20.20` (HTTP server)
  - Protokol: TCP, Portlar: `49xxx -> 8000`

### 2.4. Oqimlarning Bo‘linishi (Flow Expiry: Active Timeout va Inactive Timeout)
Oqim (Flow) tushunchasi tarmoq orqali o‘tuvchi 5 ta parametr bilan aniqlanadi (**5-tuple**):
$$\text{Flow} = \{ \text{Src IP}, \text{Dst IP}, \text{Src Port}, \text{Dst Port}, \text{Protocol} \}$$

Router xotirasi (RAM) cheklangan bo‘lgani uchun oqimlar abadiy xotirada turmaydi. Ular quyidagi 3 holatda yakunlanadi (expire bo‘ladi) va kollektorga eksport qilinadi:
1. **Inactive Timeout (Nofaollik taymauti, odatda 15 soniya):** Agar belgilangan 5-tuple bo‘yicha 15 soniya davomida birorta ham yangi paket o‘tmasa, ulanish tugagan deb hisoblanadi va oqim yozuvi eksport qilinadi.
2. **Active Timeout (Faollik taymauti, laboratoriyada `maxlife=60` s):** Agar ulanish uzluksiz davom etayotgan bo‘lsa (masalan, 10 daqiqalik katta fayl yuklanishi yoki YouTube video), router kollektorni uzoq kuttirmasligi uchun har 60 soniyada oqimni sun'iy ravishda bo‘lib, joriy hisoblagichlarni eksport qiladi va yangi hisoblagich boshlaydi. Shu sababli bitta TCP sessiyasi NetFlowda bir necha qismlarga bo‘linishi mumkin.
3. **TCP bayroqlari (FIN / RST):** Ulanish odatiy (FIN) yoki majburiy (RST) yopilganda oqim darhol yakunlanadi.

### 2.5. Wireshark Analitik Mexanizmlari
Wireshark dasturida laboratoriya davomida quyidagi vositalardan foydalaniladi:
- **Capture Filter vs Display Filter:** Capture filter (BPF) paketlarni diskka yozishdan oldin apparat/yadro darajasida filtrlaydi (ortiqcha shovqinni kamaytiradi). Display filter esa allaqachon yozib olingan PCAP fayli ichidan kerakli paketlarni ko‘rsatish uchun ishlatiladi (asl faylni o‘zgartirmaydi).
- **Statistics $\rightarrow$ Conversations:** Ikki manzil orasidagi umumiy aloqani (IP juftliklari, TCP sessiyalari, paketlar va baytlar yig‘indisi) qulay jadval ko‘rinishida jamlaydi.
- **Statistics $\rightarrow$ I/O Graphs:** Tarmoq o‘tkazuvchanligi va intensivligini vaqt o‘qi bo‘yicha vizual grafik qiladi. Masalan, bir vaqtning o‘zida HTTP trafigi (8000/TCP) bilan NetFlow eksporti (2055/UDP) o‘rtasidagi vaqt siljishini (delay) yaqqol ko‘rsatib beradi.
- **Follow $\rightarrow$ TCP Stream:** TCP ulanishidagi paketlarning ilova darajasidagi (L7) mazmunini tartiblab, to‘liq matn yoki binar ko‘rinishda tiklaydi.

---

## 3. LABORATORIYA MUHITI VA VMWARE SOZLAMALARI

> [!NOTE]
> **Foydalanuvchi Muhiti Xususiyatlari:**
> - **Asosiy tizim (Host OS):** Ubuntu Desktop (Windows EMAS!).
> - **Virtualizatsiya vositasi:** VMware Workstation (Linux versiyasi).
> - **Internet aloqasi:** Foydalanuvchining shaxsiy lokal Wi-Fi tarmog‘i (Universitet Wi-Fi emas!).
> - **Virtual mashinalar:** 3 ta Ubuntu Server 24.04 LTS (CLI rejimida).

### 3.1. Shaxsiy Wi-Fi va LAN Segment Izolyatsiyasi
Laboratoriya ishida universitet yoki uy Wi-Fi tarmog‘ining IP manzillari (masalan, `192.168.1.x` yoki `192.168.0.x`) laboratoriyadagi `10.90.10.0/24` va `10.90.20.0/24` tarmoqlari bilan to‘qnash kelmasligi kerak. 

Buning eng xavfsiz va to‘g‘ri yo‘li — **VMware LAN Segments** texnologiyasidir:
- VMware LAN Segment virtual mashinalar o‘rtasida to‘liq xususiy, apparatdan ajratilgan virtual kommutator (virtual switch) yaratadi.
- Bu tarmoqqa Host OS, tashqi Wi-Fi router yoki boshqa qurilmalar kira olmaydi. Unda tashqi DHCP ishlamaydi, bu esa bizga statik `10.90.x.x` manzillarini xatosiz va toza ishlatish kafolatini beradi.

### 3.2. VMware Workstation (Ubuntu) da Virtual Tarmoqlarni Sozlash
VMware Workstation dasturida quyidagi segmentlar yaratiladi:
1. `A9-CLIENT` — Mijoz va Router o‘rtasidagi segment.
2. `A9-SERVER` — Router va Server o‘rtasidagi segment.

#### VM Tarmoq Adapterlarini Biriktirish Tartibi:
- **CLIENT virtual mashinasi:**
  - 1 ta tarmoq adapteri: **LAN Segment** $\rightarrow$ `A9-CLIENT`
- **RTR-1 virtual mashinasi (Router):**
  - 1-tarmoq adapteri (Network Adapter 1): **LAN Segment** $\rightarrow$ `A9-CLIENT`
  - 2-tarmoq adapteri (Network Adapter 2): **LAN Segment** $\rightarrow$ `A9-SERVER`
- **SRV-MON virtual mashinasi (Server/Kollektor):**
  - 1 ta tarmoq adapteri: **LAN Segment** $\rightarrow$ `A9-SERVER`

> [!TIP]
> **Dasturlarni o‘rnatishda:** Dastlabki bosqichda paketlarni o‘rnatish uchun har bir VMda qo‘shimcha vaqtinchalik **NAT adapter** ulanadi (u sizning shaxsiy Wi-Fi orqali internetga chiqadi). O‘rnatish tugagach, VMware sozlamalarida NAT adapterining *Connected* va *Connect at power on* bayroqchalari olib tashlanadi (kabel uziladi).

### 3.3. Dasturiy Ta'minotlarni O'rnatish va Wireshark (Ubuntu Host)
VMlarda dasturlar o‘rnatiladi:

```bash
# Barcha 3 ta VMda (CLIENT, RTR-1, SRV-MON):
sudo apt update
sudo apt install -y iproute2 iputils-ping curl python3

# CLIENT VMda:
sudo apt install -y netcat-openbsd

# RTR-1 VMda:
sudo apt install -y softflowd

# SRV-MON VMda:
sudo apt install -y nfdump tcpdump
```

#### Ubuntu Hostda Wireshark O'rnatish:
Sizning asosiy kompyuteringiz Ubuntu bo‘lganligi sababli, Wiresharkni to‘g‘ridan-to‘g‘ri Ubuntu terminalida o‘rnatasiz:
```bash
sudo apt update
sudo apt install -y wireshark tshark
sudo usermod -aG wireshark $USER
```
*(O‘rnatish paytida oddiy foydalanuvchilarga paketlarni tutish ruxsatini berish so‘ralsa, `<Yes>` tanlanadi).*

### 3.4. Topologiya va IP Manzillash Jadvali

```
[ CLIENT ]                      [ RTR-1 ]                     [ SRV-MON ]
10.90.10.10/24 ------------ 10.90.10.1/24  10.90.20.1/24 ------------ 10.90.20.20/24
    (LAN_IF)                     (LAN_IF)   (SRV_IF)                      (SRV_IF)
        \                           /           \                           /
         ====== A9-CLIENT Segment ==             ====== A9-SERVER Segment ==
```

| Virtual Mashina | Interfeys roli | Namunaviy nom (VMware) | IPv4 Manzil | Shlyuz (Gateway) | Vazifasi |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **CLIENT** | A9-CLIENT adapter | `ens33` yoki `enp0s8` | `10.90.10.10/24` | `10.90.10.1` | HTTP so‘rovlar va test trafigi generatsiyasi |
| **RTR-1** | A9-CLIENT adapter | `ens33` yoki `enp0s8` | `10.90.10.1/24` | — | LAN interfeysi, NetFlow kuzatuv nuqtasi |
| **RTR-1** | A9-SERVER adapter | `ens34` yoki `enp0s9` | `10.90.20.1/24` | — | Server segmenti, NetFlow eksport manbasi |
| **SRV-MON** | A9-SERVER adapter | `ens33` yoki `enp0s8` | `10.90.20.20/24` | `10.90.20.1` | HTTP server, nfcapd kollektor, tcpdump |

> [!IMPORTANT]
> Linuxda interfeys nomlari tizim drayveriga qarab `ens33`, `ens34`, `enp0s3` yoki `enp0s8` bo‘lishi mumkin. Har bir VMda avval `ip -br link` buyrug‘ini terib, interfeysingizning asl nomini aniqlang va buyruqlardagi o‘zgaruvchilarga shu nomni kiriting!

---

## 4. MASHQNI BAJARISH BO‘YICHA QADAMMA-QADAM QO‘LLANMA

Ushbu bo‘limda barcha bosqichlar ketma-ketlikda, har bir kiritiladigan buyruq va uning har bir flagi (kaliti) nima vazifa bajarishi bilan batafsil yoritilgan.

---

### 4.1. 1-Bosqich: Tarmoq Interfeyslari va Statik Marshrutlarni Sozlash

Ushbu bosqichda uchala VMda laboratoriya interfeyslari ko‘tariladi, IP manzillar o‘rnatiladi va paketlar router orqali o‘tishi uchun statik yo‘llar (routing) belgilanadi.

#### A) CLIENT Konsolida:
Root foydalanuvchisiga o‘tamiz va tarmoqni sozlaymiz:
```bash
sudo -i
export TZ=UTC
IF=ens33
ip link set "$IF" up
ip addr replace 10.90.10.10/24 dev "$IF"
ip route replace 10.90.20.0/24 via 10.90.10.1 dev "$IF"
mkdir -p /root/A9
ip -br addr
ip route
```

🔍 **CLI Buyruqlari Tahlili:**
- `sudo -i`: Root (administrator) huquqi bilan doimiy interaktiv sessiyani ochadi. Tarmoq sozlamalarini boshqarish uchun root huquqi shart.
- `export TZ=UTC`: Tizim vaqt mintaqasini UTC (Universal Time Coordinated) qilib belgilaydi. Wireshark va log fayllaridagi vaqtlar bir xil global vaqt bilan mos kelishi uchun bu juda muhim.
- `IF=ens33`: O‘zgaruvchiga interfeys nomini yuklaydi (VMware'da odatda `ens33`). Agar `ip -br link` da boshqa nom chiqsa (masalan `enp0s8`), o‘shani yozasiz.
- `ip link set "$IF" up`: Interfeysni ma'muriy holatini faollashtiradi (link UP holatga keltiriladi).
- `ip addr replace 10.90.10.10/24 dev "$IF"`: Interfeysga `10.90.10.10` IP manzilini va `/24` (255.255.255.0) tarmoq niqobini biriktiradi. `add` o‘rniga `replace` ishlatilishi agar avval eski manzil bo‘lsa, xatolik bermay uni toza almashtirishni ta'minlaydi.
- `ip route replace 10.90.20.0/24 via 10.90.10.1 dev "$IF"`: `10.90.20.0/24` (serverlar tarmog‘i) ga boruvchi barcha paketlarni `10.90.10.1` (RTR-1 routeri) orqali jo‘natish qoidasini kiritadi.
- `mkdir -p /root/A9`: Laboratoriya fayllari, loglar va hisobotlar saqlanadigan ishchi jildni yaratadi (`-p` jild mavjud bo‘lsa xato bermaydi).
- `ip -br addr` va `ip route`: Sozlangan manzillar va yo‘nalishlar jadvalini qisqa (`-br` = brief) formatda ekranga chiqarib tekshiradi.

---

#### B) RTR-1 Konsolida:
Routerning ikkala interfeysini sozlaymiz va yadroda paketlarni marshrutlashni (packet forwarding) yoqamiz:
```bash
sudo -i
export TZ=UTC
LAN_IF=ens33
SRV_IF=ens34
ip link set "$LAN_IF" up
ip link set "$SRV_IF" up
ip addr replace 10.90.10.1/24 dev "$LAN_IF"
ip addr replace 10.90.20.1/24 dev "$SRV_IF"
sysctl -w net.ipv4.ip_forward=1
mkdir -p /root/A9
ip route
```

🔍 **CLI Buyruqlari Tahlili:**
- `LAN_IF=ens33` va `SRV_IF=ens34`: 1-adapter (Mijoz tomoni) va 2-adapter (Server tomoni) interfeys o‘zgaruvchilarini kiritish.
- `ip addr replace 10.90.10.1/24 dev "$LAN_IF"`: Mijoz segmentidagi interfeysga shlyuz IP manzilini beradi.
- `ip addr replace 10.90.20.1/24 dev "$SRV_IF"`: Server segmentidagi interfeysga shlyuz IP manzilini beradi.
- `sysctl -w net.ipv4.ip_forward=1`: Linux yadrosining (kernel) IPv4 paketlarini bir interfeysdan ikkinchisiga uzatish (routing/forwarding) funksiyasini faollashtiradi (`-w` parametrni darhol yozish). Busiz Linux router vazifasini bajarmaydi va begona paketlarni tashlab yuboradi!

---

#### C) SRV-MON Konsolida:
Server interfeysini sozlaymiz va kerakli jildlar daraxtini quramiz:
```bash
sudo -i
export TZ=UTC
IF=ens33
ip link set "$IF" up
ip addr replace 10.90.20.20/24 dev "$IF"
ip route replace 10.90.10.0/24 via 10.90.20.1 dev "$IF"
mkdir -p /root/A9/{www,out,flows}
chmod 700 /root/A9
ip route
```

🔍 **CLI Buyruqlari Tahlili:**
- `ip route replace 10.90.10.0/24 via 10.90.20.1 dev "$IF"`: Mijozlar tarmog‘iga (`10.90.10.0/24`) javob paketlarini qaytarish uchun shlyuz sifatida routerning `10.90.20.1` manzilini ko‘rsatadi.
- `mkdir -p /root/A9/{www,out,flows}`: Bir vaqtning o‘zida uchta katalog yaratadi:
  - `www`: HTTP server fayllari uchun;
  - `out`: Hisobotlar, PCAP va log fayllari uchun;
  - `flows`: Kollektor tomonidan yozib olinadigan NetFlow oqim fayllari uchun.
- `chmod 700 /root/A9`: Ushbu katalogga faqat root foydalanuvchisi kira olishi va yoza olishi uchun xavfsizlik ruxsatini belgilaydi.

---

### 4.2. 2-Bosqich: Tarmoq Bog'lanishi va Vaqtni Tekshirish

Tarmoq ulanishlarini tekshirish — amaliyotning eng muhim qadami. Agar ping o‘tmasa, keyingi bosqichlarga o‘tish mumkin emas.

#### CLIENT Konsolida:
```bash
ping -c 3 10.90.10.1
ping -c 3 10.90.20.20
ip route get 10.90.20.20
```

🔍 **CLI Buyruqlari Tahlili:**
- `ping -c 3 10.90.10.1`: CLIENT dan routergacha bo‘lgan aloqani tekshirish uchun 3 ta ICMP Echo Request paketi jo‘natadi (`-c 3` count=3). 0% packet loss bo‘lishi shart.
- `ping -c 3 10.90.20.20`: CLIENT dan oxirgi manzil (SRV-MON) gacha marshrutlash to‘g‘ri ishlayotganini tekshiradi. Paket: CLIENT $\rightarrow$ RTR-1 $\rightarrow$ SRV-MON bo‘ylab borib qaytadi.
- `ip route get 10.90.20.20`: Yadro jadvali bo‘yicha `10.90.20.20` ga boruvchi paket qaysi interfeys va qaysi shlyuz orqali chiqib ketishini simulyatsiya qilib ko‘rsatadi.

#### Har Uchala VMda Vaqtni Tekshirish:
```bash
date -u '+%Y-%m-%dT%H:%M:%SZ'
```
🔍 **CLI Buyruqlari Tahlili:**
- `date -u '+%Y-%m-%dT%H:%M:%SZ'`: Tizim vaqtini UTC formatida ISO-8601 standarti bo‘yicha chiqaradi (masalan, `2026-11-02T10:30:00Z`). Har uch VM vaqti bir xil bo‘lishi lozim, aks holda PCAP va NetFlow vaqtlari mos kelmay qoladi.

---

### 4.3. 3-Bosqich: HTTP Xizmatini Ishga Tushirish (SRV-MON)

SRV-MON serverida sinov veb-resurslari yaratiladi va Python orqali yengil HTTP server ishga tushiriladi.

```bash
cd /root/A9
printf 'A9: lokal tarmoq monitoringi sinovi\n' > www/index.html
dd if=/dev/zero of=www/payload.bin bs=1024 count=1024 status=none
python3 -m http.server 8000 --bind 10.90.20.20 \
  --directory /root/A9/www > out/http.log 2>&1 &
echo $! > /run/a9-http.pid
ss -lntp 'sport = :8000'
```

🔍 **CLI Buyruqlari Tahlili:**
- `printf '...' > www/index.html`: Kichik matnli bosh sahifa yaratadi.
- `dd if=/dev/zero of=www/payload.bin bs=1024 count=1024 status=none`:
  - `if=/dev/zero`: Nollar oqimidan o‘qiydi.
  - `of=www/payload.bin`: Chiqish fayli.
  - `bs=1024 count=1024`: 1024 bayt $\times$ 1024 marta = aynan **1 MiB (1 048 576 bayt)** hajmdagi binar test faylini hosil qiladi. Ushbu fayl katta hajmli yuklanishni (burst) tekshirish uchun zarur.
- `python3 -m http.server 8000 ... &`:
  - `-m http.server`: Python o‘rnatilgan HTTP server modulini chaqiradi.
  - `8000`: Xizmat tinglaydigan TCP port.
  - `--bind 10.90.20.20`: Server faqat laboratoriya IP manziliga kelgan so‘rovlarni qabul qiladi.
  - `--directory /root/A9/www`: Sayt fayllari joylashgan papka.
  - `> out/http.log 2>&1`: Barcha so‘rovlar jurnali (stdout va stderr) `http.log` fayliga yo‘naltiriladi.
  - `&`: Dasturni orqa fonda (background) yurgizadi.
- `echo $! > /run/a9-http.pid`: `$!` oxirgi orqa fonda ishga tushirilgan jarayon identifikatorini (PID) bildiradi. Uni keyinchalik xavfsiz to‘xtatish uchun maxsus faylga saqlab qo‘yamiz.
- `ss -lntp 'sport = :8000'`:
  - `-l`: Faqat tinglayotgan (listening) soketlar;
  - `-n`: Portlarni nom bilan emas, raqam bilan ko‘rsatish;
  - `-t`: Faqat TCP protokoli;
  - `-p`: Qaysi jarayon (PID va dastur nomi) tinglayotganini ko‘rsatish;
  - `'sport = :8000'`: Manba porti 8000 bo‘lgan holatni filtrlaydi.

**CLIENT orqali tekshirish:**
```bash
curl -f http://10.90.20.20:8000/index.html
```
*Natija: Ekranga "A9: lokal tarmoq monitoringi sinovi" chiqishi kerak.*

---

### 4.4. 4-Bosqich: NetFlow Kollektori (nfcapd) va Paket Yozuvini (tcpdump) Boshlash (SRV-MON)

> [!IMPORTANT]
> Tartib qoidasi: Dastlab kollektor (`nfcapd`) va paket tutuvchi (`tcpdump`) ishga tushirilishi kerak, shundan so‘nggina eksportchi (`softflowd`) ishga tushiriladi. Bu tartib orqali eksportchi yuboradigan dastlabki NetFlow v9 shablon paketlari PCAP fayliga ham, kollektorga ham to‘liq tushadi.

```bash
# SRV-MON konsolida:
nfcapd -D -b 10.90.20.20 -p 2055 \
  -w /root/A9/flows -t 30 -P /run/a9-nfcapd.pid

ss -lunp 'sport = :2055'

tcpdump -Z root -i "$IF" -nn -s 0 -U \
  -w /root/A9/out/A9_monitoring.pcap \
  'host 10.90.10.10 or udp port 2055' \
  > /root/A9/out/tcpdump.log 2>&1 &

echo $! > /run/a9-tcpdump.pid
sleep 2
cat /root/A9/out/tcpdump.log
```

🔍 **CLI Buyruqlari Tahlili:**
- `nfcapd`: nfdump paketidagi NetFlow oqimlarini qabul qiluvchi tinglovchi xizmat (daemon).
  - `-D`: Daemon rejimi (orqa fonda mustaqil ishlaydi).
  - `-b 10.90.20.20`: Faqat serverning `10.90.20.20` IP manziliga kelgan oqimlarni bog‘lab (bind) oladi.
  - `-p 2055`: NetFlow standart UDP porti.
  - `-w /root/A9/flows`: Oqim fayllari yoziladigan katalog.
  - `-t 30`: Oqim fayllarini har 30 soniyada yangilab (rotate qilib), diskka yozib boradi.
  - `-P /run/a9-nfcapd.pid`: Jarayon PID faylini yaratadi.
- `ss -lunp 'sport = :2055'`: UDP 2055 portida `nfcapd` muvaffaqiyatli tinglayotganini tasdiqlaydi (`-u` UDP).
- `tcpdump`: Linuxdagi eng mashhur paket tutuvchi dastur.
  - `-Z root`: tcpdump paketlarni yozish paytida ruxsatlarini oddiy foydalanuvchiga tushirmasdan, `root` huquqini saqlab qoladi (`/root` katalogiga to‘siqsiz yoza olishi uchun).
  - `-i "$IF"`: Qaysi interfeysdagi paketlarni tutish (masalan `ens33`).
  - `-nn`: IP manzillar va portlarni DNS yoki xizmat nomiga aylantirmasdan sof raqam ko‘rinishida saqlaydi (resurs tejaydi va aniqlik beradi).
  - `-s 0`: Snapshot length = 0 (cheksiz). Har bir paketning barcha baytlarini boshidan oxirigacha to‘liq yozib oladi (hech qanday payload kesilmaydi).
  - `-U`: Paket tutuvchi buferini paket kelishi bilan darhol diskka yozadi (packet-buffered).
  - `-w /root/A9/out/A9_monitoring.pcap`: Barcha tutib olingan xom paketlarni Wireshark formatidagi `.pcap` fayliga yozadi.
  - `'host 10.90.10.10 or udp port 2055'`: **BPF Capture Filter**. Faqat mijoz (`10.90.10.10`) bilan bog‘liq paketlarni hamda UDP 2055 dagi NetFlow eksport paketlarini yozib oladi. Boshqa ortiqcha tizim paketlari e'tiborga olinmaydi.
- `sleep 2 && cat /root/A9/out/tcpdump.log`: 2 soniya kutib, log faylini ko‘rsatadi. Logda `listening on ens33...` yozuvi paydo bo‘lishi lozim.

---

### 4.5. 5-Bosqich: NetFlow Eksportchisini (softflowd) Sozlash va Boshlash (RTR-1)

Routerning faqat `LAN_IF` (mijoz tomoni) interfeysida eksportchini ishga tushiramiz. 

> [!WARNING]
> Nega ikkala interfeysda emas, faqat `LAN_IF` da? Chunki agar ikkala interfeysda ham softflowd yoqilsa, mijozdan serverga o‘tayotgan bitta tranzit paket kirishda ham, chiqishda ham hisoblanib, hisobotda trafik 2 barobar ko‘payib (dublikat bo‘lib) ko‘rinadi!

```bash
# RTR-1 konsolida:
softflowd -i "$LAN_IF" -n 10.90.20.20:2055 -v 9 \
  -p /run/a9-softflowd.pid -c /run/a9-softflowd.ctl \
  -t maxlife=60

softflowctl -c /run/a9-softflowd.ctl timeouts
softflowctl -c /run/a9-softflowd.ctl statistics
```

🔍 **CLI Buyruqlari Tahlili:**
- `softflowd`: Linux interfeysini tinglab, NetFlow eksport qiluvchi dastur.
  - `-i "$LAN_IF"`: Kuzatuv ostiga olinadigan interfeys (`ens33`).
  - `-n 10.90.20.20:2055`: Tayyor bo‘lgan oqimlarni qabul qiluvchi kollektor manzili va porti.
  - `-v 9`: NetFlow protokoli versiyasi 9 (v9 shablonli).
  - `-p /run/a9-softflowd.pid`: Jarayon PID fayli.
  - `-c /run/a9-softflowd.ctl`: Jarayonni boshqarish uchun UNIX domain socket nazorat fayli.
  - `-t maxlife=60`: Active timeout qiymatini 60 soniya qilib belgilaydi.
- `softflowctl -c /run/a9-softflowd.ctl timeouts`: Joriy taymautlar parametrlarini tekshiradi (active, inactive taymautlar).
- `softflowctl -c /run/a9-softflowd.ctl statistics`: Eksportchi statistikasini (kuzatilgan paketlar, yaratilgan oqimlar soni) ko‘rsatadi.

---

### 4.6. 6-Bosqich: Nazoratli Trafik Hosil Qilish (CLIENT)

CLIENT mashinasida ikkita alohida ssenariy bo‘yicha nazoratli trafik hosil qilinadi:
1. **Baza bosqichi (Baseline):** Tinch, odatiy tarmoq harakati (ping, 3 ta kichik sahifa so‘rovi, 1 ta katta fayl yuklanishi).
2. **Yuklanish bosqichi (Burst) va portlar tekshiruvi:** Qisqa vaqt ichida tezkor takroriy yuklanish (10 marta katta fayl so‘rovi) va 3 ta yopiq portga ulanishga urinish.

```bash
# CLIENT konsolida:
cd /root/A9

# 1-Bosqich: BASELINE TRAFIK
{ date -u '+BASELINE_START %Y-%m-%dT%H:%M:%SZ'
  ping -c 4 10.90.20.20
  for i in 1 2 3; do
    curl -f -sS -o /dev/null \
      "http://10.90.20.20:8000/index.html?phase=base&i=$i"
    sleep 2
  done
  curl -f -sS -o /dev/null \
    http://10.90.20.20:8000/payload.bin?phase=base
  date -u '+BASELINE_END %Y-%m-%dT%H:%M:%SZ'
} 2>&1 | tee baseline.txt

sleep 5

# 2-Bosqich: BURST TRAFIK VA PORTLARNI TEKSHIRISH
{ date -u '+BURST_START %Y-%m-%dT%H:%M:%SZ'
  for i in $(seq 1 10); do
    curl -f -sS -o /dev/null \
      "http://10.90.20.20:8000/payload.bin?phase=burst&i=$i"
    sleep 1
  done
  for port in 81 82 83; do
    nc -vz -w 1 10.90.20.20 "$port"
  done
  date -u '+BURST_END %Y-%m-%dT%H:%M:%SZ'
} 2>&1 | tee burst.txt
```

🔍 **CLI Buyruqlari Tahlili:**
- `{ ... } 2>&1 | tee baseline.txt`: Blok ichidagi barcha buyruqlar natijasi va vaqt belgilarini ekranga chiqarish bilan bir vaqtda `baseline.txt` fayliga yozib oladi.
- `ping -c 4 10.90.20.20`: Baza bosqichida 4 ta ICMP paket jo‘natadi.
- `curl -f -sS -o /dev/null ...`:
  - `-f` (fail): HTTP xato kodlarida jim qoladi.
  - `-sS` (silent/show-error): Ortiqcha yuklanish foizini ko‘rsatmaydi, lekin xato bo‘lsa chiqaradi.
  - `-o /dev/null`: Yuklab olingan faylni diskka saqlamaydi (tizimni to‘ldirmaslik uchun axlatga tashlaydi), lekin tarmoqda real trafik to‘liq o‘tadi!
  - `?phase=base&i=$i`: URI orqali qaysi bosqichdagi so‘rov ekanligini Wiresharkda aniq ajratish imkonini beradi.
- `sleep 2` va `sleep 1`: So‘rovlar orasida tanaffus qiladi.
- `nc -vz -w 1 10.90.20.20 "$port"`:
  - `nc` (netcat): Portlarga TCP ulanish o‘rnatuvchi vosita.
  - `-v`: Verbose (batafsil ma'lumot berish).
  - `-z`: Zero-I/O rejimi (faqat port ochiq yoki yopiqligini tekshiradi, ma'lumot jo‘natmaydi).
  - `-w 1`: Taymaut 1 soniya.
  - 81, 82, 83-portlarda hech qanday xizmat ishlamagani uchun server operatsion tizimi darhol **TCP RST/ACK** javobini qaytaradi (`Connection refused`). Bu Wiresharkda yopiq portlar anomaliyasini isbotlaydi.

Jami kutiladigan natija: 14 ta HTTP GET (3 ta index.html + 1 ta baseline payload.bin + 10 ta burst payload.bin).

---

### 4.7. 7-Bosqich: Oqimlarni Yakunlash, Saqlash va Jarayonlarni To‘xtatish

Eksportchi xotirasida (cache) qolib ketgan oxirgi oqimlarni majburiy chiqarib, kollektorga jo‘natish va barcha dasturlarni to‘g‘ri (graceful) to‘xtatish kerak.

#### A) RTR-1 Konsolida:
```bash
softflowctl -c /run/a9-softflowd.ctl dump-flows > /root/A9/active-flows.txt
softflowctl -c /run/a9-softflowd.ctl send-template
softflowctl -c /run/a9-softflowd.ctl expire-all
softflowctl -c /run/a9-softflowd.ctl statistics > /root/A9/exporter-statistics.txt
softflowctl -c /run/a9-softflowd.ctl shutdown
```

🔍 **CLI Buyruqlari Tahlili:**
- `dump-flows > active-flows.txt`: Routerning joriy xotirasidagi oqimlar holatini matn fayliga saqlaydi.
- `send-template`: **O‘ta muhim buyruq!** Kollektorga v9 shablonini yana bir bor yuborishni buyuradi.
- `expire-all`: Routerning barcha taymautlarini nolga tenglashtirib, xotiradagi har bir faol oqimni darhol kollektorga NetFlow paketi ko‘rinishida yuboradi. (Hech qachon `delete-all` qilmang, u eksport qilmasdan o‘chirib yuboradi!).
- `shutdown`: `softflowd` jarayonini to‘g‘ri to‘xtatadi.

---

#### B) SRV-MON Konsolida:
Kollektorga paketlar yetib kelishi uchun 5 soniya kutamiz, so‘ng jarayonlarni to‘xtatamiz:
```bash
sleep 5
kill -TERM "$(cat /run/a9-nfcapd.pid)"
sleep 2
kill -INT "$(cat /run/a9-tcpdump.pid)"
sleep 2

cat /root/A9/out/tcpdump.log
ls -lh /root/A9/flows /root/A9/out/A9_monitoring.pcap
nfdump -r /root/A9/flows -o long 'host 10.90.10.10' > /root/A9/out/flows-all.txt
cat /root/A9/out/flows-all.txt
```

🔍 **CLI Buyruqlari Tahlili:**
- `kill -TERM "$(cat /run/a9-nfcapd.pid)"`: `nfcapd` kollektoriga SIGTERM signali yuboradi. Natijada kollektor ochiq turgan joriy oqim faylini (current buffer) butun holatda diskka muhrlaydi va yopadi. (Hech qachon `kill -9` ishlatmang, aks holda oxirgi oqimlar buzilib qoladi).
- `kill -INT "$(cat /run/a9-tcpdump.pid)"`: `tcpdump` ga SIGINT (Ctrl+C ekvivalenti) signali yuboradi. tcpdump barcha qolgan paketlarni PCAP fayliga yozib, xulosa statistikasini logga chiqaradi.
- `cat /root/A9/out/tcpdump.log`: Tutib olingan paketlar sonini va eng asosiysi **"0 packets dropped by kernel"** (yadro tomonidan yo‘qotilgan paketlar soni nol) ekanligini tasdiqlaydi.
- `nfdump -r /root/A9/flows -o long 'host 10.90.10.10'`: Kollektor yozgan oqimlarni o‘qiydi va umumiy oqimlar ro‘yxatini chiqaradi.

---

### 4.8. 8-Bosqich: Dalillarni Ubuntu Hostga Ko‘chirish

Barcha to‘plangan dalillarni bitta arxivga yig‘amiz va Ubuntu Host tizimiga ko‘chiramiz:

#### SRV-MON Konsolida Arxivlash:
```bash
cd /root/A9
sha256sum out/A9_monitoring.pcap > out/SHA256SUMS.txt
tar -czf /root/A9/A9_dalillar.tar.gz out flows
```
🔍 **CLI Buyruqlari Tahlili:**
- `sha256sum ... > out/SHA256SUMS.txt`: PCAP faylining nazorat xesh-yig‘indisini hisoblaydi. Bu fayl ko‘chirish paytida shikastlanmaganini (yaxlitligini) isbotlaydi.
- `tar -czf A9_dalillar.tar.gz out flows`: `out` va `flows` kataloglarini bitta siqilgan `.tar.gz` arxiviga birlashtiradi (`-c` create, `-z` gzip, `-f` file).

#### Ubuntu Hostga Ko‘chirish Usullari (Sizning Ubuntu muhitingiz uchun):

*1-usul: SCP orqali (Eng qulay va tezkor usul)*
Ubuntu Host terminalidan turib SRV-MON serveriga ulanib, fayllarni to‘g‘ridan-to‘g‘ri yuklab olish mumkin (agar SSH o‘rnatilgan bo‘lsa):
```bash
# Ubuntu Host terminalida:
scp root@10.90.20.20:/root/A9/out/A9_monitoring.pcap ~/Downloads/
scp root@10.90.20.20:/root/A9/A9_dalillar.tar.gz ~/Downloads/
```

*2-usul: Python yengil HTTP yuklovchi orqali*
Agar tarmoq segmentlari xususiy bo‘lsa, SRV-MONda vaqtincha papkani internet/hostga ochish yoki VMware Shared Folder (`vmhgfs-fuse`) dan foydalanish mumkin:
```bash
# SRV-MON da VMware Shared Folderni ulash:
mkdir -p /mnt/a9share
vmhgfs-fuse .host:/A9 /mnt/a9share -o allow_other
cp /root/A9/out/A9_monitoring.pcap /mnt/a9share/
cp /root/A9/A9_dalillar.tar.gz /mnt/a9share/
```

---

### 4.9. 9-Bosqich: Wireshark Yordamida Paketlarni Tahlil Qilish

Ubuntu Hostda Wiresharkni ochamiz:
```bash
wireshark A9_monitoring.pcap &
```

Wireshark menyusidan: **View $\rightarrow$ Time Display Format $\rightarrow$ UTC Date and Time of Day** (`2026-11-02 10:35:12`) formatini tanlang.

Quyidagi jadval bo‘yicha Display filtrlarni kiritib, kerakli dalillarni skrinshot olasiz:

| № | Vazifa | Wireshark Display Filter | Nima tekshiriladi / Dalil | Skrinshot belgisi |
| :--- | :--- | :--- | :--- | :--- |
| 1 | HTTP GET so‘rovlari | `http.request && ip.src == 10.90.10.10` | Pastki holat panelidagi **Displayed** qiymati (aynan **14** ta bo‘lishi kerak). | **W1** |
| 2 | Tanlangan TCP Sessiyasi | `ip.addr == 10.90.10.10 && tcp.port == 8000` | **Statistics $\rightarrow$ Conversations $\rightarrow$ TCP**. Bitta ulanishning paketlari, baytlari va vaqti. | **W2** |
| 3 | Trafik va Eksport Grafigi | Graf 1: `tcp.port == 8000 && ip.addr == 10.90.10.10`<br>Graf 2: `udp.port == 2055` | **Statistics $\rightarrow$ I/O Graphs**. Vaqt intervali: 1 sek, Y Axis: Packets. HTTP va NetFlow cho‘qqilarining vaqt bo‘yicha siljishi. | **W3** |
| 4 | NetFlow v9 Dekodlash | `cflow` | Header (Version 9), Template ID va uning ichidagi Data Record (Src/Dst IP, Portlar, Hisoblagichlar). | **W4** |
| 5 | Yopiq Portlar Javobi | `tcp.flags.reset == 1 && ip.src == 10.90.20.20` | Serverdan 81, 82, 83-portlarga kelgan **RST, ACK** paketlari. | — |

> [!TIP]
> Agar Wireshark 8000-portni HTTP sifatida yoki 2055-portni NetFlow sifatida avtomatik tanimasa:
> Paket ustida o‘ng tugmani bosing $\rightarrow$ **Decode As...** $\rightarrow$ TCP port 8000 ni `HTTP` ga, UDP port 2055 ni esa `CFLOW` ga o‘zgartiring.

---

### 4.10. 10-Bosqich: nfdump Yordamida Oqimlar Hisobotini Olish (SRV-MON)

SRV-MON konsolida kollektor yozgan oqimlarni tahlil qilish uchun quyidagi 3 ta hisobot buyrug‘i bajariladi:

```bash
cd /root/A9

# 1. Bitta 5-tuple bo'yicha agregatsiyalangan HTTP oqimlari:
nfdump -r flows -a -o long \
  'host 10.90.10.10 and port 8000 and proto tcp' \
  > out/http-aggregated.txt

# 2. Eng ko'p bayt uzatilgan maqsad portlari (Top Talkers / Top Destination Ports):
nfdump -r flows -s dstport/bytes -n 10 \
  'src ip 10.90.10.10 and proto tcp' > out/top-dstports.txt

# 3. Mashinada o'qiladigan CSV hisoboti:
nfdump -r flows -o csv 'host 10.90.10.10' > out/flows.csv

cat out/http-aggregated.txt
cat out/top-dstports.txt
```

🔍 **CLI Buyruqlari Tahlili:**
- `nfdump -r flows`: `flows` katalogidagi barcha NetFlow binar fayllarini o‘qiydi.
- `-a`: **Aggregation**. Taymautlar sababli bo‘lingan oqimlarni bir xil 5 ta belgi (Src IP, Dst IP, Src Port, Dst Port, Protocol) bo‘yicha yagona qatorga birlashtiradi.
- `-o long`: Natijani kengaytirilgan formatda (boshlanish va tugash vaqti, davomiyligi, protokoli, IP manzillar, bayroqlar, paketlar va baytlar soni) chiqaradi.
- `-s dstport/bytes`: Statistik saralash buyrug‘i. Trafikni maqsad portlari bo‘yicha guruhlab, eng ko‘p bayt o‘tgan portlarni kamayish tartibida joylashtiradi.
- `-n 10`: Faqat eng yuqori 10 ta qatorni (Top 10) ko‘rsatadi. (Bu yerda 8000-port birinchi o‘rinda, 81, 82, 83 portlari esa juda kam paket/bayt bilan ko‘rinadi).
- `-o csv`: Natijalarni Excel, Pandas yoki SIEM tizimlariga import qilish uchun qulay CSV formatida saqlaydi.

---

### 4.11. 11-Bosqich: Bitta Ulanishni Wireshark va NetFlowda O'zaro Solishtirish

Ushbu qadam laboratoriya ishining kulminatsion nuqtasi hisoblanadi. Wiresharkdan bitta to‘liq HTTP TCP sessiyasini tanlaymiz va uning parametrlarini NetFlow bilan solishtiramiz.

1. Wiresharkda `http.request` filtrini qo‘ying va masalan birinchi `payload.bin` so‘rovini toping.
2. Uning mijoz portini (Src Port) aniqlang (masalan, `49152`).
3. SRV-MONda shu aniq port bo‘yicha NetFlow oqimini chiqaramiz:

```bash
# SRV-MON konsolida (49152 o'rniga o'zingizdagi mijoz portini yozasiz):
CLIENT_PORT=49152
FLOW_FILTER="src ip 10.90.10.10 and dst ip 10.90.20.20 and proto tcp and src port $CLIENT_PORT and dst port 8000"
nfdump -r /root/A9/flows -a -o long "$FLOW_FILTER" > /root/A9/out/selected-flow.txt
cat /root/A9/out/selected-flow.txt
```

4. Wiresharkda ham faqat shu bir yo‘nalish uchun filtr kiriting:
`ip.src == 10.90.10.10 && ip.dst == 10.90.20.20 && tcp.srcport == 49152 && tcp.dstport == 8000`

#### Qiyosiy Solishtirish Jadvali:
*(Ushbu jadval hisobotga kiritiladi va olingan haqiqiy raqamlar bilan to‘ldiriladi)*

| Ko‘rsatkich | Wireshark / PCAP | NetFlow (nfdump) | Farq va Ilmiy Izoh |
| :--- | :--- | :--- | :--- |
| **Manba IP va Port** | `10.90.10.10:49152` | `10.90.10.10:49152` | Bir xil (Mijozning vaqtinchalik efemer porti). |
| **Maqsad IP va Port** | `10.90.20.20:8000` | `10.90.20.20:8000` | Bir xil (HTTP xizmati tinglayotgan port). |
| **Boshlanish vaqti (UTC)** | `HH:MM:SS.xxx` | `HH:MM:SS.xxx` | Sekund darajasida to‘liq mos keladi. |
| **Tugash vaqti (UTC)** | `HH:MM:SS.xxx` | `HH:MM:SS.xxx` | Sessiya yopilgan vaqt. |
| **Paketlar soni (Packets)** | Masalan: 12 paket | Masalan: 12 paket | Bir xil yo‘nalish bo‘yicha paketlar soni mos keladi. |
| **Baytlar hajmi (Bytes)** | Masalan: 864 bayt | Masalan: 720 bayt | **Farq sababi:** Wireshark Ethernet (L2) freym hajmini (14 baytlik sarlavha + FCS) hisoblaydi. NetFlow esa IP (L3) sarlavhasidan boshlab hisoblaydi! |

---

### 4.12. 12-Bosqich: Xavfsizlik Hodisasini Tahlil Qilish va Monitoring Qoidasini Ishlab Chiqish

#### Tahliliy Xulosa:
Laboratoriya davomida olingan dalillar asosida quyidagi xavfsizlik xulosasi shakllantiriladi:
- **Hodisa tahlili:** CLIENT `burst` fazasida qisqa vaqt ichida ketma-ket 10 marta 1 MiB hajmdagi faylni yuklab oldi (jami 10 MiB dan ortiq). Shuningdek, 81, 82, 83 portlariga ulanishga urinib, serverdan RST/ACK javoblarini oldi.
- **Dalillar:**
  1. `http.log` — bitta IP dan soniyasiga 1 tadan ortiq so‘rov kelganini ko‘rsatadi;
  2. Wireshark `I/O Graphs` — trafik grafigida keskin cho‘qqi (burst spike) hosil bo‘lganini ko‘rsatadi;
  3. `top-dstports.txt` — 8000-port asosiy trafikni olganini, lekin 81, 82, 83 portlarida shubhali muvaffaqiyatsiz ulanishlar izi qolganini tasdiqlaydi.

#### Tavsiya Etiladigan Monitoring Qoidasi (Detection Rule):
> **Tavsiya etiladigan SIEM / Snort / Suricata qoidasi:**
> *"Agar bitta ichki manba IP manzilidan 8000/TCP portiga 15 soniya ichida 5 tadan ortiq yangi TCP ulanishi o‘rnatilsa yoki umumiy yuklangan ma'lumot hajmi 8 MiB dan oshsa, hamda ayni manba ketma-ket yopiq portlarga murojaat qilsa, ushbu holat 'Shubhali ma'lumot sizdirilishi (Data Exfiltration) yoki DoS/Bruteforce urinishi' sifatida belgilanib, SOC xavfsizlik tahlilchisiga ALARM xabari yuborilsin."*

---

## 5. NOSOZLIKLARNI BARTARAF ETISH VA OG‘ZAKI HIMOYA

### 5.1. Amaliyotda Uchraydigan Xatoliklar va Ularning Yechimlari

1. **Ping o‘tmayapti (`Destination Host Unreachable` yoki 100% loss):**
   - *Tekshirish:* `ip -br link` buyrug‘i orqali interfeyslar `UP` ekanligini ko‘ring.
   - *Tekshirish:* RTR-1 da `sysctl net.ipv4.ip_forward` qiymati `1` ekanligini tasdiqlang.
   - *Tekshirish:* CLIENT va SRV-MON da qarama-qarshi tarmoqqa marshrut (`ip route`) router IP si orqali kiritilganini tekshiring.
   - *Tekshirish:* VMware sozlamalarida virtual adapterlar to‘g‘ri LAN Segmentga (`A9-CLIENT` va `A9-SERVER`) ulanganini qayta tekshiring.

2. **HTTP xizmati ochilmayapti (`curl: (7) Failed to connect`):**
   - *Tekshirish:* SRV-MON da `ss -lntp 'sport = :8000'` qiling. Agar chiqmasa, Python server to‘xtab qolgan. `cat /root/A9/out/http.log` orqali xatoni ko‘ring va Python buyrug‘ini qayta yurgizing.

3. **PCAP da NetFlow (UDP 2055) paketlari ko‘rinmayapti:**
   - *Tekshirish:* RTR-1 da `softflowctl -c /run/a9-softflowd.ctl statistics` buyrug‘ini bering. Yaratilgan va eksport qilingan oqimlar sonini ko‘ring.
   - *Tekshirish:* `softflowctl expire-all` buyrug‘ini bering (chunki oqimlar faollik taymauti tugamaguncha router xotirasida turishi mumkin).

4. **nfdump bo‘sh natija qaytaryapti (`No matching flows`):**
   - *Tekshirish:* `ls -la /root/A9/flows` orqali `nfcapd.YYYYMMDD...` fayllari yaratilganini tekshiring.
   - *Tekshirish:* `nfcapd` hali joriy faylni yopmagan bo‘lishi mumkin. Unga `kill -TERM <pid>` yuborib, buferni diskka yozishga majburlang.

---

### 5.2. Og'zaki Himoya Savollari va Ilmiy-Amaliy Javoblar

#### 1. Capture filter va Display filter orasidagi farq nima?
**Javob:** Capture filter (BPF) paketlar drayver va yadro darajasida ushlanayotgan paytda ishlaydi. U faqat filtrga mos paketlarni diskka yozadi, qolganlarini tashlab yuboradi (fayl hajmini tejaydi, ammo tushib qolgan ma'lumotni qayta tiklab bo‘lmaydi). Display filter esa allaqachon diskka yozilgan PCAP fayli ustida tahlil paytida ishlaydi, u fayl tarkibini o‘zgartirmaydi, faqat kerakli paketlarni ekranda ko‘rsatadi.

#### 2. NetFlow eksportchisi va kollektorining vazifalari qanday?
**Javob:** Eksportchi (masalan, marshrutizator yoki `softflowd`) bevosita trafik o‘tadigan interfeysda turadi, paketlar sarlavhalarini o‘qiydi, 5-tuple bo‘yicha oqimlar jadvalini tuzadi va hisoblagichlarni yuritadi. Taymaut tugagach, bu ma'lumotlarni NetFlow protokolida tarmoq orqali uzatadi. Kollektor (`nfcapd`) esa markaziy serverda turib, bir yoki bir nechta eksportchilardan kelgan UDP paketlarni qabul qiladi, yaxlitligini tekshiradi va binar fayllarga arxivlaydi.

#### 3. NetFlow v9 shabloni yo‘qolsa, data yozuvini tahlil qilishga nima bo‘ladi?
**Javob:** Data yozuvlarini tahlil qilib (decode) bo‘lmaydi. Chunki NetFlow v9 da ma'lumotlar qat'iy tuzilishga ega emas, har bir maydonning turi va uzunligi (masalan, IPv4 4 bayt, Port 2 bayt) oldindan yuborilgan Shablon (Template) orqali belgilanadi. Shablon bo‘lmasa, kelgan baytlar qatorini qaysi biri IP, qaysi biri port yoki bayt hisoblagichi ekanini ajratib bo‘lmaydi.

#### 4. Nega eksport paketidagi tashqi IP bilan ichidagi flow IP bir xil emas?
**Javob:** Tashqi IP sarlavhasi — bu NetFlow telemetriyasini olib ketayotgan transport paketining manzillari (Manba: Router `10.90.20.1`, Maqsad: Kollektor `10.90.20.20`, UDP 2055). Ichki flow ma'lumotlari esa — router interfeysidan o‘tgan kuzatuv ostidagi real mijoz trafigining parametrlari (Manba: `10.90.10.10`, Maqsad: `10.90.20.20`, TCP 8000).

#### 5. Nega bitta TCP ulanishidan bir necha oqim yozuvi chiqishi mumkin?
**Javob:** Birinchidan, NetFlow bir yo‘nalishli (unidirectional) hisoblanadi: so‘rov yo‘nalishi (Mijoz $\rightarrow$ Server) alohida oqim, javob yo‘nalishi (Server $\rightarrow$ Mijoz) alohida oqim bo‘ladi. Ikkinchidan, agar TCP sessiyasi Active timeout (masalan, 60 soniya) dan uzoq davom etsa, eksportchi uni har 60 soniyada bo‘lib, alohida yozuv sifatida eksport qiladi.

#### 6. Nega HTTP trafik bilan eksport UDP trafigining vaqt grafigi farq qiladi?
**Javob:** HTTP trafigi real vaqtda — mijoz so‘rov yuborgan soniyada darhol sodir bo‘ladi. NetFlow eksport trafigi (UDP 2055) esa kechikish bilan paydo bo‘ladi, chunki eksportchi trafik o‘tishi bilan darhol paket jo‘natmaydi, balki oqim tugashini (Inactive timeout) yoki faollik taymauti (Active timeout) yetib kelishini kutadi.

#### 7. Wireshark va NetFlow baytlarini tenglashtirishdan oldin nimalar tekshiriladi?
**Javob:** Avvalo, ikkala manbada aynan bir xil 5-tuple (IPlar, portlar, protokol) va bir xil yo‘nalish tanlangani tekshiriladi. Ikkinchidan, qatlam farqi (Wiresharkda L2 Ethernet freymi o‘lchanadimi yoki L3 IP paketimi) aniqlanadi. Uchinchidan, faollik taymauti sababli bo‘lingan NetFlow oqimlari `nfdump -a` yordamida birlashtirilishi kerak. To‘rtinchidan, tarmoqda paket yo‘qolishi (packet drop) bo‘lmagani tasdiqlanishi shart.

#### 8. Yopiq portning RST javobi va firewall timeouti qanday farqlanadi?
**Javob:** Agar port yopiq bo‘lsa va u yerda xizmat ishlamayotgan bo‘lsa, operatsion tizim steki darhol faol javob sifatida **TCP RST (Reset)** yoki ICMP Port Unreachable paketini qaytaradi (mijozda darhol "Connection refused" xatosi chiqadi). Agar o‘rtada Firewall (xavfsizlik devori) bo‘lsa va u paketlarni `DROP` qilsa, hech qanday javob qaytmaydi va mijoz ulanish taymauti tugaguncha kutib turadi ("Connection timed out").

#### 9. Faollik ko‘tarilishi (traffic spike) hujum ekanligini aniqlash uchun yana qanday kontekst kerak?
**Javob:** Faqatgina baytlar sonining ko‘payishi hujumni isbotlamaydi (bu qonuniy katta fayl yuklanishi, zaxira nusxalash yoki dastur yangilanishi bo‘lishi mumkin). Hujumni tasdiqlash uchun: so‘rovlar soni va chastotasi, HTTP holat kodlari (200, 404, 500), foydalanuvchi identifikatori (User-Agent), so‘ralgan URI parametrlari, manba IP reputatsiyasi hamda bir vaqtning o‘zida yopiq portlarga skanerlash mavjudligi kabi qo‘shimcha kontekstlar tahlil qilinadi.

#### 10. Trafik HTTPS (TLS) bo‘lsa, mazmun va oqim ko‘rsatkichlarining qaysilari ko‘rinadi?
**Javob:** HTTPS bo‘lganda to‘liq paket tahlilida (Wireshark) payload shifrlangan bo‘ladi (GET so‘rovi, URL manzillari, veb-sahifa mazmuni ko‘rinmaydi), faqat TLS Handshake dagi SNI (Server Name Indication) va sertifikat ko‘rinishi mumkin. NetFlow oqim tahlilida esa deyarli hech narsa yo‘qolmaydi: manba va maqsad IP manzillar, 443/TCP porti, o‘tgan baytlar va paketlar soni, vaqt davomiyligi o‘zgarishsiz to‘liq ko‘rinadi.

---

## 6. FOYDALANILGAN VA TAVSIYA ETILADIGAN MANBALAR

1. **Ubuntu 24.04 Manpages — softflowd(8):**
   *Trafikni kuzatish, oqimlarni shakllantirish va NetFlow eksport parametrlari rasmiy qo‘llanmasi.*
   Havola: [https://manpages.ubuntu.com/manpages/noble/man8/softflowd.8.html](https://manpages.ubuntu.com/manpages/noble/man8/softflowd.8.html)
2. **Ubuntu 24.04 Manpages — softflowctl(8):**
   *Eksportchini boshqarish, statistikani ko‘rish, shablon yuborish va oqimlarni chiqarish.*
   Havola: [https://manpages.ubuntu.com/manpages/noble/man8/softflowctl.8.html](https://manpages.ubuntu.com/manpages/noble/man8/softflowctl.8.html)
3. **Ubuntu 24.04 Manpages — nfcapd(1):**
   *NetFlow kollektori sozlamalari, aylantirish (rotation) va buferlash qoidalari.*
   Havola: [https://manpages.ubuntu.com/manpages/noble/man1/nfcapd.1.html](https://manpages.ubuntu.com/manpages/noble/man1/nfcapd.1.html)
4. **Ubuntu 24.04 Manpages — nfdump(1):**
   *Oqim fayllarini filtrlash, agregatsiya qilish va statistik hisobotlar yaratish vositasi.*
   Havola: [https://manpages.ubuntu.com/manpages/noble/man1/nfdump.1.html](https://manpages.ubuntu.com/manpages/noble/man1/nfdump.1.html)
5. **Wireshark User’s Guide — Display Filters:**
   *Paketlarni tahlil qilishda ifodalar va sintaksis qoidalari.*
   Havola: [https://www.wireshark.org/docs/wsug_html_chunked/ChWorkDisplayFilterSection.html](https://www.wireshark.org/docs/wsug_html_chunked/ChWorkDisplayFilterSection.html)
6. **Wireshark User’s Guide — Conversations:**
   *Tarmoq muloqotlari statistikasi va tahlili.*
   Havola: [https://www.wireshark.org/docs/wsug_html_chunked/ChStatConversations.html](https://www.wireshark.org/docs/wsug_html_chunked/ChStatConversations.html)
7. **IETF RFC 3954 — Cisco Systems NetFlow Services Export Version 9:**
   *NetFlow v9 protokoli spetsifikatsiyasi, shablonlar arxitekturasi va formatlari.*
   Havola: [https://datatracker.ietf.org/doc/html/rfc3954](https://datatracker.ietf.org/doc/html/rfc3954)
8. **IETF RFC 7011 — Specification of the IP Flow Information Export (IPFIX) Protocol:**
   *NetFlow v9 asosida yaratilgan zamonaviy xalqaro oqim standarti.*
   Havola: [https://datatracker.ietf.org/doc/html/rfc7011](https://datatracker.ietf.org/doc/html/rfc7011)
9. **Wireshark Rasmiy Yuklab Olish Sahifasi:**
   Havola: [https://www.wireshark.org/download.html](https://www.wireshark.org/download.html)
