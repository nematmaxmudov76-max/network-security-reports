# A4 TARMOQ XAVFSIZLIGI LABORATORIYA ISHI BO‘YICHA YAKUNIY XULOSA VA TAHLILNOMA

Mazkur hujjat `description.md` va `A4_Tarmoq_xavfsizligi_VMware_proxy.pdf` fayllarida keltirilgan amaliy vazifalarni to‘liq tahlil qilib, laboratoriya ishining tub mohiyati, ishlatilgan barcha vositalar, CLI komandalar ketma-ketligi, kasbiy o‘sish darajalari hamda amaliy muhandislik tavsiyalarini qamrab oladi.

---

## 1. VAZIFANI BAJARISHDAN ASOSIY MAQSAD

Laboratoriya mashg‘ulotining bosh strategik maqsadi — Linux operatsion tizimi yadrosi (Kernel space) darajasida ishlovchi **Netfilter** quyi tizimi va zamonaviy **nftables** vositasi yordamida **holatli xavfsizlik devori (Stateful Firewall)**, **zonal marshrutlash (Zonal Routing)** hamda **manzillarni tarjima qilish (NAT — Network Address Translation)** arxitekturasini noldan loyihalash, sozlash, sinash va dalillash ko‘nikmalarini shakllantirishdir.

### Asosiy funksional va xavfsizlik maqsadlari:
1. **Zonal Izolyatsiya va Dasturiy Marshrutlash (L3 Routing):**
   - Tarmoqni uchta o‘zaro izolyatsiyalangan segmentga ajratish:
     - `A4-LAN` (`10.40.10.0/24`) — Ichki xavfsiz korporativ foydalanuvchilar tarmog‘i;
     - `A4-DMZ` (`10.40.30.0/24`) — Ommaviy xizmatlar joylashgan demilitarizatsiyalangan oraliq zona;
     - `A4-WAN` (`198.18.0.0/24`) — Tashqi, ishonchsiz global tarmoq (Internet simulyatsiyasi, RFC 2544).
   - Ubuntu Server 24.04 negizidagi `FW-RTR` mashinasini 3 ta alohida virtual interfeys bilan jihozlab, Linux yadrosida IPv4 paketlar tranzitini (`net.ipv4.ip_forward=1`) faollashtirish.

2. **"Default Deny" (Nolga asoslangan ishonch / Zero Trust) Konsepsiyasi:**
   - Firewallning kiruvchi (`input`) va tranzit (`forward`) zanjirlarida sukut bo‘yicha barcha harakatlarni qat’iy taqiqlash (`policy drop`).
   - Faqat xizmat zarurati bo‘lgan, manba, maqsad, protokol va porti qat’iy belgilangan oqimlargagina minimal ruxsat berish (Least Privilege tamoyili).

3. **Holatli Nazorat (Stateful Connection Tracking):**
   - Paketlarni bir-biridan uzilgan holda (stateless) emas, balki ularning ulanish holatlari kontekstida (`conntrack`: `NEW`, `ESTABLISHED`, `RELATED`, `INVALID`) filtrlash.
   - Ichkaridan boshlangan qonuniy sessiyalarning qaytuvchi javob paketlarini bitta umumiy va o‘ta xavfsiz `ct state established,related accept` qoidasi orqali avtomatik o‘tkazish.

4. **NAT Texnologiyalarini Amaliy Joriy Etish:**
   - **SNAT (Source NAT):** Ichki LAN mijozlari (`10.40.10.0/24`) tashqi manbalarga murojaat qilganda, ularning xususiy IP manzillarini router WAN interfeysidagi yagona ochiq IP (`198.18.0.1`) ga almashtirib chiqarish (`postrouting` zanjirida).
   - **DNAT (Destination NAT / Port Forwarding):** Tashqi tarmoqdagi mijozlar DMZ serverining ichki IP manzilini bilmagan holda, router WAN IP si va noan’anaviy portiga (`198.18.0.1:8080`) murojaat qilganda so‘rovni DMZ dagi veb-serverga (`10.40.30.20:80`) yo‘naltirish (`prerouting` zanjirida).

5. **Kiberxavfsizlik Monitoringi va Himoya:**
   - Taqiqlangan ulanish urinishlarini tizim jurnallariga (`journalctl / dmesg`) yozish.
   - Jurnallar orqali tizim diskini to‘ldirib yuborishga qaratilgan DoS hujumlarining oldini olish uchun log yozish tezligini cheklash (**Rate-limiting**: `limit rate 5/minute`).

6. **Daliliy Isbotlash Madaniyati (Forensic Verification):**
   - Shunchaki "sayt ochildi yoki ochilmadi" deyish bilan cheklanmasdan, qoidalarning ishlashini 4 qatlamli dalillar bilan isbotlash:
     - `nftables counter` (paket va bayt hisoblagichlari);
     - `journalctl` (tizim xavfsizlik loglari);
     - `tcpdump` / `.pcap` (paket darajasidagi sarlavhalar tahlili);
     - `conntrack` (Linux yadrosi sessiyalar xotirasi holati).

7. **Nazoratli Xatolik (Fault Injection) Tahlili:**
   - Stateful qoidani ataylab o‘chirib, TCP 3 bosqichli qo‘l berishish (Handshake) jarayonining aynan qaytuvchi `SYN-ACK` fazasida qanday uzilishini amalda tadqiq qilish.

---

## 2. ISHLATILGAN TASHQI KUTUBXONALAR VA QO‘SHIMCHA TOOL'LAR

Laboratoriya muhitini qurish, tarmoqni boshqarish, xavfsizlik devorini dasturlash va tahlil qilish uchun quyidagi dasturiy vositalar va utilitalar ishlatildi:

| Kategoriya | Dastur / Paket nomi | Asosiy roli va funksiyasi |
| :--- | :--- | :--- |
| **Virtualizatsiya** | **VMware Workstation Pro / Player** | Izolyatsiyalangan virtual mashinalar yaratish, `LAN Segment` virtual kommutatorlarini qurish va tizim holatini saqlash (`Snapshot`). |
| **Arxivator** | **p7zip-full (`7z`)** | Tayyor siqilgan `.7z` virtual mashina obrazlarini diskka to‘liq ochish. |
| **Tizim va Yadro** | **systemd / hostnamectl** | Xizmatlarni (SSH, Nginx, nftables) boshqarish va VM hostnamelarini bir xil tartibga keltirish. |
| **Yadro Parametrlari** | **procps (`sysctl`)** | Linux yadrosining tarmoq steki parametrlarini o‘zgartirish (`net.ipv4.ip_forward=1`). |
| **Tarmoq va Marshrutlash** | **iproute2 (`ip`)** | Interfeyslarni yoqish/o‘chirish (`ip link`), eski manzillarni tozalash (`ip addr flush`), statik IP biriktirish (`ip addr add`) va shlyuzlarni boshqarish (`ip route`). |
| **Soket Diagnostikasi** | **iproute2 (`ss`)** | Tizimda qaysi portlar tinglayotganini (Listening) aniqlash (`ss -lntp`). |
| **Xavfsizlik Devori** | **nftables (`nft`)** | Netfilter asosidagi zamonaviy paket filtrlovchi, qoidalar to‘plamini kompilyatsiya qiluvchi va yadroga yuklovchi asosiy xavfsizlik utilitasi. |
| **Sessiyalar Tahlili** | **conntrack-tools (`conntrack`)** | Linux yadrosining faol ulanishlar holati jadvalini (Connection Tracking Table) o‘qish va tekshirish. |
| **Paket Tahlilchisi** | **tcpdump (libpcap kutubxonasi)** | Tarmoq interfeyslaridan o‘tayotgan IP paketlarni real vaqtda ushlash, BPF filtrlash va `.pcap` faylga yozish. |
| **HTTP Mijoz** | **curl** | Veb-so‘rovlar yuborish, sarlavhalarni tekshirish (`-I`), proksini aylanib o‘tish (`--noproxy`) va javob vaqtini cheklash (`--max-time`). |
| **Port Prober / Skaner** | **netcat-openbsd (`nc`)** | Ixtiyoriy TCP portlarning ochiq yoki yopiqligini ma’lumot yubormasdan tekshirish (`nc -vz`). |
| **ICMP Diagnostika** | **iputils-ping (`ping`)** | 3-darajali bog‘lanish (L3 Reachability) va ICMP qoidalarini tekshirish. |
| **Veb Server (DMZ)** | **nginx** | DMZ zonasida haqiqiy ishlab chiqarish darajasidagi HTTP xizmatini taqdim etish (80-port). |
| **Masofaviy Boshqaruv** | **openssh-server (`sshd`)** | Router va DMZ da boshqaruv porti (22-port) xizmatini ko‘rsatish va unga ruxsat/taqiq siyosatini sinash. |
| **Sinov Serveri (WAN)** | **python3 (`http.server` moduli)** | EXT-1 mashinasida ruxsat berilgan (8000) va taqiqlangan (9000) sinov portlarini tezkor ko‘tarish. |
| **Matn va Oqim Utilitalari**| **coreutils (`tee`, `timeout`, `awk`, `grep`)** | Chiqishlarni fayllarga yozish, buyruqlar bajarilish vaqtini cheklash va matnlarni filtrlash. |

---

## 3. QADAMMA-QADAM CLI BUYRUQLAR VA ULARNING QO‘LLANILISH MAQSADI

Har bir bosqichda berilgan buyruqlar, ularning kalitlari (flaglari) va nima maqsadda bajarilgani xronologik tartibda quyidagicha kechadi:

```
+---------------------------------------------------------------------------------------------------------+
|                                  LABORATORIYA AMALIYOT ZANJIRI                                          |
|                                                                                                         |
|  [0. Arxiv & VM] -> [1. Paketlar & Snap] -> [2. FW-RTR Sozlash] -> [3. Mijozlar Sozl.]                |
|           |                                                                                             |
|           v                                                                                             |
|  [4. Servislar (Nginx/Python)] -> [5. nftables Qoidalari] -> [6. Sintaksis & Audit]                    |
|           |                                                                                             |
|           v                                                                                             |
|  [7. T1-T10 Testlari] -> [8. tcpdump & conntrack Dalillash] -> [9. Nazoratli Xato] -> [10. Saqlash]   |
+---------------------------------------------------------------------------------------------------------+
```

### 0-BOSQICH: Host Muhitini Tayyorlash
```bash
sudo apt update && sudo apt install -y p7zip-full
```
* **Maqsadi:** Ubuntu host kompyuterida siqilgan `.7z` arxivlarni ochish uchun yuqori unumdorlikka ega arxivator dasturini o‘rnatish.
```bash
mkdir -p ~/A4-LAB/{FW-RTR,LAN-CLIENT,WEB-DMZ,EXT-1,Report}
7z x UbuntuServer_24.04_VM.7z -o~/A4-LAB/FW-RTR/
cp -r ~/A4-LAB/FW-RTR/* ~/A4-LAB/LAN-CLIENT/
cp -r ~/A4-LAB/FW-RTR/* ~/A4-LAB/WEB-DMZ/
cp -r ~/A4-LAB/FW-RTR/* ~/A4-LAB/EXT-1/
```
* **Maqsadi:** Har bir virtual tugun uchun alohida katalog yaratish va tayyor virtual disk nusxalarini joylashtirish.
* *Muhim detal:* VMware da ilk bor yoqilganda **"I Copied It"** tanlanadi, bu har bir VM ga unikal MAC manzil berilishini kafolatlaydi.

---

### 1-BOSQICH: Paketlarni O‘rnatish va Boshlang‘ich Holatni Saqlash
Har bir VMda o‘z nomini belgilash:
```bash
sudo hostnamectl set-hostname FW-RTR       # (Tegishli VMda LAN-CLIENT, WEB-DMZ, EXT-1)
```
* **Maqsadi:** Tarmoq identifikatsiyasini to‘g‘rilash va buyruqlar qaysi mashinada bajarilayotganini chalkashtirib yubormaslik.

Paketlarni o‘rnatish (vaqtinchalik NAT adapteri orqali):
```bash
# FW-RTR da:
sudo apt install -y nftables conntrack tcpdump openssh-server iproute2
sudo systemctl enable --now ssh

# LAN-CLIENT da:
sudo apt install -y curl netcat-openbsd iputils-ping tcpdump iproute2

# WEB-DMZ da:
sudo apt install -y nginx openssh-server curl iputils-ping tcpdump
sudo systemctl enable --now nginx ssh

# EXT-1 da:
sudo apt install -y python3 curl netcat-openbsd tcpdump iproute2
```
* **Maqsadi:** Har bir mashinaga o‘z vazifasiga mos xizmatlar, diagnostika va tahlil utilitalarini yuklash.
* **So‘nggi qadam:** Mashinalar o‘chiriladi (`sudo poweroff`), vaqtinchalik NAT uziladi, faqat tegishli LAN segmentlar qoldirilib, **`A4-base`** nomli toza snapshot olinadi.

---

### 2-BOSQICH: FW-RTR Interfeyslari va Marshrutlashni Sozlash
```bash
ip -br link
for i in /sys/class/net/en*; do echo "$i -> $(cat "$i/address")"; done
```
* **Maqsadi:** FW-RTR dagi 3 ta tarmoq kartasining MAC manzillarini ko‘rib, qaysi interfeys qaysi segmentga (`ens37` = LAN, `ens38` = DMZ, `ens39` = WAN) tegishli ekanini aniqlash.

Statik IP manzillar biriktirish:
```bash
for IFACE in ens37 ens38 ens39; do
    sudo ip addr flush dev "$IFACE"
    sudo ip link set "$IFACE" up
done

sudo ip addr add 10.40.10.1/24 dev ens37
sudo ip addr add 10.40.30.1/24 dev ens38
sudo ip addr add 198.18.0.1/24 dev ens39
```
* `ip addr flush dev <nom>` — DHCP dan qolgan tasodifiy manzillarni tozalaydi.
* `ip link set <nom> up` — Portlarni faol (UP) holatga keltiradi.
* `ip addr add <IP/mask>` — Belgilangan gateway statik IP larini interfeyslarga bog‘laydi.

Yadro darajasida marshrutlashni (IP Forwarding) yoqish:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-a4-router.conf
```
* **Maqsadi:** Linux yadrosiga paketlarni bir interfeysdan boshqasiga o‘tkazishga (router sifatida ishlashga) ruxsat berish va rebootdan so‘ng ham ushbu parametrni saqlab qolish.

---

### 3-BOSQICH: Mijoz Mashinalari Tarmoq Sozlamalari
```bash
# LAN-CLIENT:
sudo ip addr flush dev ens33 && sudo ip link set ens33 up
sudo ip addr add 10.40.10.10/24 dev ens33
sudo ip route replace default via 10.40.10.1 dev ens33

# WEB-DMZ:
sudo ip addr flush dev ens33 && sudo ip link set ens33 up
sudo ip addr add 10.40.30.20/24 dev ens33
sudo ip route replace default via 10.40.30.1 dev ens33

# EXT-1:
sudo ip addr flush dev ens33 && sudo ip link set ens33 up
sudo ip addr add 198.18.0.10/24 dev ens33
sudo ip route del default 2>/dev/null || true
```
* `ip route replace default via <GW>` — Barcha tashqi trafikni FW-RTR tomon yo‘naltiruvchi shlyuzni o‘rnatadi.
* `ip route del default` (EXT-1 da) — EXT-1 dan shlyuzni o‘chiradi. Bu juda muhim: EXT-1 private `10.40.10.0/24` tarmog‘ini bilmasligi kerak. Shundagina SNAT ishlagani aniq isbotlanadi.

---

### 4-BOSQICH: Sinov Xizmatlarini Ko‘tarish
WEB-DMZ da:
```bash
echo '<h1>A4 DMZ service</h1><p>10.40.30.20:80</p>' | sudo tee /var/www/html/index.html
sudo systemctl reload nginx
sudo ss -lntp | grep -E ':(22|80)\b'
curl --noproxy "*" http://127.0.0.1/
```
* **Maqsadi:** Nginx veb-sahifasini sozlash, 80 va 22 portlar ochiqligini (`ss -lntp`) va xizmat ishlayotganini lokal tekshirish.

EXT-1 da:
```bash
nohup python3 -m http.server 8000 --bind 198.18.0.10 > ~/A4_ext_8000.log 2>&1 &
nohup python3 -m http.server 9000 --bind 198.18.0.10 > ~/A4_ext_9000.log 2>&1 &
ss -lntp | grep -E ':(8000|9000)\b'
```
* **Maqsadi:** EXT-1 da bitta ruxsat beriladigan (8000) va bitta firewall tomonidan bloklanadigan (9000) xizmatlarni fon rejimida (`nohup`) ko‘tarish.

---

### 5-BOSQICH: FW-RTR da nftables Rulesetini Yozish
Qoidalar yaxlit holda `/etc/nftables.conf` ga yoziladi:
```bash
sudo tee /etc/nftables.conf >/dev/null <<'EOF'
#!/usr/sbin/nft -f
flush ruleset

define LAN_IF = "ens37"
define DMZ_IF = "ens38"
define WAN_IF = "ens39"
define LAN_NET = 10.40.10.0/24
define DMZ_WEB = 10.40.30.20
define WAN_IP  = 198.18.0.1

table inet a4_filter {
    chain input {
        type filter hook input priority filter; policy drop;
        iifname "lo" counter accept
        ct state invalid counter drop
        ct state established,related counter accept
        iifname $LAN_IF ip saddr $LAN_NET tcp dport 22 counter accept
        iifname $LAN_IF ip saddr $LAN_NET icmp type echo-request counter accept
        limit rate 5/minute log prefix "A4-INPUT-DROP " flags all
        counter drop
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state invalid counter drop
        ct state established,related counter accept
        iifname $LAN_IF oifname $WAN_IF ip saddr $LAN_NET tcp dport 8000 counter accept
        iifname $LAN_IF oifname $DMZ_IF ip saddr $LAN_NET ip daddr $DMZ_WEB tcp dport 80 counter accept
        iifname $WAN_IF oifname $DMZ_IF ip daddr $DMZ_WEB tcp dport 80 ct status dnat counter accept
        limit rate 5/minute log prefix "A4-FWD-DROP " flags all
        counter drop
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}

table ip a4_nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        iifname $WAN_IF ip daddr $WAN_IP tcp dport 8080 counter dnat to 10.40.30.20:80
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname $WAN_IF ip saddr $LAN_NET counter snat to 198.18.0.1
    }
}
EOF
```
* **Maqsadi:** "Default Deny" xavfsizlik modeliga binoan faqat ruxsat etilgan P1–P8 siyosatlarini kiritish, SNAT va DNAT ni sozlash.

---

### 6-BOSQICH: Sintaksis Tekshiruvi, Qoidalarni Qo‘llash va Audit
```bash
sudo nft -c -f /etc/nftables.conf
sudo nft -f /etc/nftables.conf
sudo nft -a list table inet a4_filter
sudo nft -a list table ip a4_nat
sudo systemctl enable nftables
```
* `nft -c -f <fayl>` — Yadroga yuklamasdan avval sintaktik xatolarni tekshiradi (`$?` kodi `0` bo‘lishi shart).
* `nft -f <fayl>` — Qoidalar to‘plamini yadroga atomik tarzda yuklaydi.
* `nft -a list ...` — Qoidalarning identifikatorlari (`handle`) bilan birga ko‘rsatadi.

---

### 7-BOSQICH: 10 ta Ijobiy va Salbiy Testlarni O‘tkazish (T1–T10)
```bash
# LAN-CLIENT da:
ping -c 2 -W 2 10.40.10.1                                   # T1: LAN -> FW Ping [Ruxsat]
nc -vz -w 3 10.40.10.1 22                                   # T2: LAN -> FW SSH [Ruxsat]
curl --noproxy "*" --max-time 5 -I http://198.18.0.10:8000/ # T4: LAN -> EXT:8000 (SNAT orqali) [Ruxsat]
curl --noproxy "*" --max-time 5 http://198.18.0.10:9000/    # T5: LAN -> EXT:9000 [Bloklangan]
curl --noproxy "*" --max-time 5 http://10.40.30.20/         # T6: LAN -> DMZ HTTP:80 [Ruxsat]
nc -vz -w 3 10.40.30.20 22                                  # T7: LAN -> DMZ SSH:22 [Bloklangan]

# EXT-1 da:
nc -vz -w 3 198.18.0.1 22                                   # T3: EXT -> FW SSH:22 [Bloklangan]
curl --noproxy "*" --max-time 5 http://198.18.0.1:8080/     # T8: EXT -> FW:8080 (DNAT orqali) [Ruxsat]
curl --noproxy "*" --max-time 5 http://198.18.0.1:80/       # T9: EXT -> FW:80 to'g'ridan-to'g'ri [Bloklangan]

# WEB-DMZ da:
ping -c 2 -W 2 10.40.10.10                                  # T10: DMZ -> LAN Ping [Bloklangan]
```
Natijalarni FW-RTR da ko‘rish:
```bash
sudo nft -a list chain inet a4_filter input
sudo nft -a list chain inet a4_filter forward
sudo journalctl -k -n 40 --no-pager | grep 'A4-'
```
* **Maqsadi:** Har bir oqimning xavfsizlik talablariga mosligini tekshirish. Ruxsat berilganlar bo‘yicha `counter` oshadi, bloklanganlar esa `A4-INPUT-DROP` yoki `A4-FWD-DROP` log yozuvini hosil qiladi.

---

### 8-BOSQICH: Paket Darajasida Tahlil: tcpdump va conntrack Dalillari
1. **SNAT isbotlash:**
   ```bash
   # EXT-1 da:
   sudo timeout 15 tcpdump -ni ens33 'tcp port 8000 and host 198.18.0.1' -c 6
   # LAN-CLIENT da:
   curl --noproxy "*" --max-time 5 http://198.18.0.10:8000/
   ```
   * **Natija:** EXT-1 dagi tcpdump paket LAN (`10.40.10.10`) dan emas, router WAN IP si (`198.18.0.1`) dan kelganini ko‘rsatadi.

2. **DNAT isbotlash:**
   ```bash
   # FW-RTR da (WAN interfeys):
   sudo tcpdump -ni ens39 'host 198.18.0.10 and tcp port 8080'
   # FW-RTR da (DMZ interfeys):
   sudo tcpdump -ni ens38 'host 10.40.30.20 and tcp port 80'
   # EXT-1 da:
   curl --noproxy "*" --max-time 5 http://198.18.0.1:8080/
   ```
   * **Natija:** WAN ga kelgan `198.18.0.1:8080` paketi routerdan DMZ ga o‘tayotganda `10.40.30.20:80` ga aylangani tasdiqlanadi.

3. **PCAP yozish va conntrack tahlili:**
   ```bash
   sudo timeout 20 tcpdump -Z root -ni any '(host 198.18.0.10 or host 10.40.30.20)' -w ~/A4_nat.pcap &
   sudo conntrack -L -p tcp | grep -E '10.40.10.10|10.40.30.20|198.18.0.10' | tee ~/A4_conntrack.txt
   ```
   * **Maqsadi:** Xom paketlarni Wireshark formatida saqlash va conntrack xotirasida `src`, `dst`, `sport`, `dport` teskari juftliklari qanday eslab qolinganini qayd etish.

---

### 9-BOSQICH: Nazoratli Xatolik (Fault Injection) Tahlili
Stateful qoidani olib tashlash:
```bash
HANDLE=$(sudo nft -a list chain inet a4_filter forward | awk '/ct state established,related/{print $NF; exit}')
sudo nft delete rule inet a4_filter forward handle "$HANDLE"
```
LAN-CLIENT da ulanishni tekshirish:
```bash
curl --noproxy "*" --no-keepalive --max-time 5 http://198.18.0.10:8000/
echo "Chiqish kodi: $?"
```
* **Kuzatuv:** Buyruq 5 soniya qotib turadi va `exit code 28` (Connection timed out) beradi.
* **Sababi:** Mijozning birinchi `SYN` paketi forward dan o‘tadi, ammo EXT-1 qaytargan `SYN-ACK` javob paketi forward zanjirida `established,related` qoidasi yo‘qligi sababli `policy drop` ga uchraydi. TCP sessiyasi o‘rnatilmaydi.

Qoidani xavfsiz qayta tiklash:
```bash
sudo nft -c -f /etc/nftables.conf && sudo nft -f /etc/nftables.conf
```
* **Maqsadi:** Yaxlit toza konfiguratsiya faylini qayta yuklash orqali tarmoq ishini bir lahzada qayta tiklash.

---

## 4. SELF-STUDY: BU BILIMLARNI QAYSI DARAJAGA YETGANDA O‘RGANISH KERAK?

Mazkur laboratoriya ishi oddiy boshlang‘ich dasturlash yoki umumiy kompyuter savodxonligi darsi emas. U professional **Tarmoq Xavfsizligi (Network Security)**, **DevOps/DevSecOps** va **Linux Tizim Muhandisligi (System Engineering)** sohalarining markaziy qismiga to‘g‘ri keladi.

O‘zingiz mustaqil o‘rganayotganingizda (Self-study) ushbu bilimlarni o‘zlashtirish bosqichlari quyidagi darajalar ierarxiyasida ko‘rinadi:

```
               [ 5-DARAJA: SecOps, Kernel & eBPF Engineering ]
                                      ^
                                      |
         [ 4-DARAJA: Cloud Infrastructure & Next-Gen Enterprise Security ]
                                      ^
                                      |
=====>   [ 3-DARAJA: ADVANCED LINUX NETWORKING & STATEFUL FIREWALLING ]   <===== (BIZNING ISH)
                                      ^
                                      |
                [ 2-DARAJA: Linux Admin & Basic Networking (CCNA) ]
                                      ^
                                      |
                      [ 1-DARAJA: Fundamental IT Asoslari ]
```

### Darajalar tafsiloti:

* **1-daraja: Fundamental IT va Kompyuter Tarmoqlari Asoslari (Prerequisites / Boshlang‘ich)**
  * *O‘rganiladigan mavzular:* OSI 7 qatlamli modeli, TCP/IP steki, IPv4 adreslash va subnetting (`/24`, `/16`), MAC manzillar, asosiy CLI komandalar (`ls`, `cd`, `grep`, `mkdir`).
  * *Xulosa:* Bu bosqichda siz hali nftables qoidalarini yozishga tayyor emassiz, avval paket nima ekanligini tushunish kerak.

* **2-daraja: Linux Administratorligi va Asosiy Tarmoq Boshqaruvi (Junior SysAdmin / CCNA Level)**
  * *O‘rganiladigan mavzular:* Linuxda tarmoq interfeyslarini boshqarish (`iproute2`: `ip link`, `ip addr`, `ip route`), DNS va DHCP ishlashi, `systemd` xizmatlarini boshqarish, Nginx veb-serverini sozlash, statik routing tushunchasi.
  * *Xulosa:* Bu darajaga yetganingizda, bugungi laboratoriyamizning dastlabki 1–4 bosqichlarini (tarmoqni sozlash, IP ulash, xizmat ko‘tarish) erkin tushunasiz.

* **3-daraja: Chuqurlashtirilgan Linux Tarmog‘i, Netfilter va Xavfsizlik Devorlari (Junior–Mid Security / DevOps)**
  * **Aynan biz bajargan ish ushbu darajaga to‘g‘ri keladi!**
  * *O‘rganish vaqti:* Siz Linuxda erkin ishlab, tarmoq marshrutlarini tushunganingizdan so‘ng va tarmoq xavfsizligiga chuqurlashishni boshlagan paytingizda (odatda universitetning 3-kursi yoki Junior SysAdmin/DevOps sifatida 6-12 oylik tajribadan so‘ng).
  * *O‘rganiladigan mavzular:* Netfilter arxitekturasi, Packet Flow (Prerouting -> Input/Forward -> Postrouting), Stateful vs Stateless farqi, Connection Tracking (`conntrack`), NAT turlari (SNAT, DNAT, Masquerade), Default Deny siyosati, `tcpdump` yordamida paketlarni ushlash va Wireshark tahlili.

* **4-daraja: Bulut Xavfsizligi va Korporativ Himoya (Mid–Senior Cloud Security / DevSecOps)**
  * *O‘rganiladigan mavzular:* AWS Security Groups / VPC Network ACLs, GCP Firewall Rules, Kubernetes NetworkPolicy (Calico, Cilium), Next-Generation Firewall (Palo Alto, Fortinet), IDS/IPS (Suricata, Snort), DDoS hujumlariga qarshi SYN-proxy va ratelimit strategiyalari.
  * *Bog‘liqligi:* 4-darajadagi barcha zamonaviy Cloud xavfsizlik devorlari va konteyner xavfsizlik tizimlari aynan biz o‘rgangan Netfilter va `conntrack` mantig‘iga asoslangan!

---

## 5. SHAXSIY KASBIY MASLAHATLARIM VA TAVSIYALAR

Ushbu laboratoriyadan keyin o‘z bilim va amaliyotingizni haqiqiy ishlab chiqarish (production) darajasiga ko‘tarish uchun 7 ta qat’iy tavsiya:

1. **Netfilter "Packet Flow" xaritasini yoddan tasavvur qiling:**
   Qoidalarni shunchaki yodlamang. Har safar paket kelganda uning marshrutini ko‘z oldingizga keltiring:
   `Kirdi -> Prerouting (DNAT shu yerda!) -> Routing Decision -> Input (o'ziga) yoki Forward (tranzit) -> Output -> Postrouting (SNAT shu yerda!) -> Chiqdi`. Bu xaritani bilgan mutaxassis hech qachon "DNAT qildim, nega sayt ochilmayapti?" deb adashmaydi (chunki forwarddagi ruxsatni unutmagan bo‘ladi).

2. **Har doim paket darajasida dalil ko‘rishni odat qiling:**
   IT muhandisligida "Menda ishlamayapti" degan so‘z yo‘q. Faqat `tcpdump`, `curl -v` va `conntrack` dalillari bor. Har doim bir tomondan so‘rov yuborib, ikkinchi tomonda `tcpdump -ni <int> port <port>` qilib turing. Muammoning 99% sababi paketning yo bormaganida, yoki javob qaytayotganda yadro tomonidan tashlab yuborilganida (`drop`) bo‘ladi.

3. **"Default Deny" mentalitetini shakllantiring:**
   Tajribasiz ma’murlar avval hamma narsani ochib yuborib, keyin faqat ma’lum portlarni bloklaydi (`blacklist`). Bu xavfli xato! Haqiqiy professional tizimda har doim barcha eshiklar qulflanadi (`policy drop`), so‘ngra faqat zarur bo‘lgan oqimlargagina minimal ruxsat beriladi (`whitelist`).

4. **Avtomatlashtirishni (Infrastructure as Code) o‘rganing:**
   Ushbu laboratoriyada 4 ta VMni qo‘lda sozlash, IP larni qo‘lda kiritish bir necha soat vaqt oladi. Ertaga self-study jarayonida ushbu topologiyani **Vagrant**, **Ansible** yoki **Docker/Containerlab** orqali bitta buyruq (`vagrant up`) bilan avtomatik ko‘tariladigan skript yozishga harakat qiling. Bu sizni DevOps darajasiga olib chiqadi.

5. **Ishlab chiqarish (Production) xavflarini doim inobatga oling:**
   - Hech qachon real serverda rate-limitsiz log yozmang (`limit rate 5/minute`). Aks holda hujumchi diskingizni bir necha daqiqada to‘ldirib, tizimni qulatadi.
   - Conntrack xotirasi chegaralangan (`sysctl net.netfilter.nf_conntrack_max`). Yuqori yuklamali (High-load) tizimlarda ushbu jadval to‘lib qolmasligini nazorat qilish lozim.

6. **Intervyu va himoyalarda "Nega?" savoliga mexanizm bilan javob bering:**
   O‘qituvchi yoki ish beruvchi "Nega `established,related` kerak?" deb so‘rasa, "Javob qaytishi uchun" deb qisqa aytmang. "Chunki TCP 3-bosqichli qo‘l berishishda mijoz SYN yuboradi, server qaytargan SYN-ACK paketi esa firewall forward zanjirida ESTABLISHED/RELATED holatida bo‘ladi. Agar bu qoida bo‘lmasa, qaytuvchi paket default deny ga uchrab drop bo‘ladi va sessiya uziladi" deb javob bersangiz, sizning darajangiz darhol yuqori baholanadi.

7. **nftables evolyutsiyasini kuzatib boring:**
   Eski qo‘llanmalardagi `iptables` endi tarixga aylanmoqda. Sizning eng zamonaviy `nftables` ni o‘rganguningiz katta ustunlik. Keyingi qadam sifatida Linux yadrosining eng ilg‘or texnologiyasi bo‘lgan **eBPF (Extended Berkeley Packet Filter)** va **XDP (eXpress Data Path)** texnologiyalarini o‘rganishni reja qiling.

---

## 6. XULOSA

Xulosa qilib aytganda, `description.md` va taqdim etilgan A4 laboratoriya topshirig‘i tarmoq arxitekturasi va kiberxavfsizlikning eng asosiy poydevorini qamrab olgan. 

Ushbu amaliyot orqali:
- Tarmoq marshrutlashi (Routing), paketlarni filtrlash (Firewalling) va manzil tarjimasi (NAT) alohida, lekin Netfilter yadrosida bir-biri bilan uzviy bog‘liq qatlamlar ekani;
- Stateful firewalling orqali qaytuvchi javob trafigini o‘ta xavfsiz boshqarish mumkinligi;
- NAT jarayoni faqat birinchi paketda ulanish yozuvini yaratib, keyingi barcha oqimlarni conntrack xotirasi orqali avtomatik qayta ishlashi;
- Har qanday xavfsizlik devori konfiguratsiyasi albatta paketlar ushlovchisi (`tcpdump`), holatlar jurnali (`conntrack`) va hisoblagichlar (`counter`) orqali dalillanishi shartligi to‘liq isbotlandi.
