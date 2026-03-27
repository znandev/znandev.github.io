title: 'Dijamin Paham! Inilah Cara Build Private DNS Server Di VPS Menggunakan AdGuard Home + Docker'
date: 2026-03-26
cover: /img/adguard-home-dns-server.png
top_img: /img/default.png
tags: [Networking, Server, DNS]
---
## 👋🏻Welcome!

![AdGuard: Local DNS Resolver with upstream DNS servers through DNS over HTTPS ( DoH )](/img/adguard-home-dns-server.png)

Halo visitians, gimana kabarnya ? Semoga sehat selalu ya. Kali ini saya ingin share cara bangun DNS Server sendiri dengan AdGuardHome menggunakan Docker. Pertama-tama izinkan saya menjelaskan apa itu AdGuardHome. AdGuard Home adalah perangkat lunak (software) open-source yang berfungsi sebagai server DNS lokal (self-hosted) dengan kemampuan untuk melakukan pemfilteran lalu lintas DNS secara real-time. Secara teknis, AdGuard Home bekerja dengan cara menerima permintaan DNS dari perangkat dalam suatu jaringan, kemudian:

- Menganalisis permintaan tersebut berdasarkan daftar filter (blocklist/allowlist)
- Memblokir domain yang teridentifikasi sebagai iklan, pelacak (tracker), atau berbahaya
- Meneruskan permintaan yang aman ke server DNS upstream (seperti Cloudflare atau Google DNS)
- Mengembalikan hasil resolusi ke perangkat pengguna

### Instalasi AdGuard Home di VPS

#### Persyaratan:
1. Kamu harus punya VPS dengan spec berikut (bisa beli di Vultr, BiznetGio, Herza dll)
- CPU  : 1 core
- RAM  : 1GB
- OS   : Debian/Ubuntu
- Port : 80, 443, 53, 784, 853 dan 3000 (tcp dan udp)
2. Domain (opsional jika ingin gunakan DoH/DoT)

#### Persiapan

```
sudo apt update && apt upgrade -y
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
```

Cek

```
docker --version
docker-compose --version
```

#### ⚠️ Matikan DNS Default (Penting!)

Untuk hindari conflict port 53, matikan DNS Default. Cek apakah digunakan:

```
sudo apt install lsof
sudo lsof -i :53
```

Jika benar digunakan silakan stop
```
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

Edit nameserver ke 1.1.1.1
```
nano /etc/resolv.conf
```

```
nameserver 1.1.1.1
```

#### Setup directory project
```
sudo mkdir -p /opt/adguardhome
cd /opt/adguardhome
sudo chown -R $USER:$USER /opt/adguardhome
nano docker-compose.yml
```
Isi dengan :
```
version: '3.8'

services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped

    ports:
      - "53:53/tcp"
      - "53:53/udp"     
      - "80:80/tcp"     # Admin UI 
      - "443:443/tcp"   # HTTPS
      - "853:853/tcp"   # DNS-over-TLS
      - "784:784/udp"   # DNS-over-QUIC
      - "3000:3000/tcp" # Init port 

    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf

    environment:
      - TZ=Asia/Jakarta
```
Start pull docker image
```
docker-compose up -d
```

Cek:
```
docker ps
```
Open link:
http://IP-VPS:3000 kemudian atur interface, username dan password.

> Note: Jangan lupa pastikan port 3000 tidak konflik ya, misal ada Grafana dll.

### Filtering

Untuk filtering iklan, tracker dan malware. Saya sudah siapkan blacklist berikut untuk dipakai 😄:
```
filters:
  - enabled: true
    url: https://adguardteam.github.io/HostlistsRegistry/assets/filter_1.txt
    name: AdGuard DNS filter
    id: 1
  - enabled: true
    url: https://big.oisd.nl
    name: OISD Big
    id: 1772749062
  - enabled: true
    url: https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt
    name: Adguard-Pro
    id: 1772749064
  - enabled: true
    url: https://malware-filter.gitlab.io/malware-filter/urlhaus-filter-agh.txt
    name: Anti-malware-01
    id: 1772749065
  - enabled: true
    url: https://phishing.army/download/phishing_army_blocklist_extended.txt
    name: Anti-malware-02
    id: 1772749066
  - enabled: true
    url: https://urlhaus.abuse.ch/downloads/hostfile/
    name: Anti-malware-03
    id: 1772749067
  - enabled: true
    url: https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/ultimate.txt
    name: Brutall-Ads
    id: 1772749069
```
Untuk langsung setting ke adguardhome, silakan edit config adguardhome di `/opt/adguardhome/conf/AdGuardHome.yaml`.
Ganti bagian `filters` di `AdGuardHome.yaml` dengan filters diatas.

Restart `adguardhome` supaya ter-apply.
```
sudo docker restart adguardhome
```

### Konfigurasi Upstream DNS & DNSSEC

Konfigurasi ini bertujuan agar menstabilkan request dan mengenskripsi traffic DNS.

#### Upstream DNS
Silakan copy Upstream DNS berikut:
```
https://dns.cloudflare.com/dns-query
https://dns.google/dns-query
```
Buka `Setting` > `DNS Settings` > `Upstream DNS` (pastekan dikolom Upstream DNS) kemudian `Apply`.

#### DNSSEC
Masih di `DNS Settings` scroll kebawah dibagian `DNS server configuration` centang `Enable DNSSEC` kemudian `Save`.

### Setup HTTPS, DNS-over-HTTPS & DNS-over-TLS

Selanjutnya kita akan lanjut untuk setup HTTPS, DoH dan DoT. Bertujuan untuk membuat DNS server kita bisa diakses secara aman, terenkripsi, dan dari jaringan mana pun tanpa batasan 😄

#### Pointing domain ke VPS

Agar DNS Server kamu bisa resolve dan terbaca oleh certbot silakan pointing terlebih dahulu domain kamu ke IP VPS.
`A record → dns.domainkamu.com → IP_VPS`

#### Generate SSL Let's Encrypt via Certbot

Install certbot terlebih dahulu:
```
sudo apt update
sudo apt install cerbot -y
```

Karena adguardhome running di port 80, kita harus stop terlebih dahulu agar certbot dapat verifikasi dan generate SSL.

```
sudo docker stop adguardhome
```

Lanjut generate SSL:
```
sudo certbot certonly --standalone -d dns.domainkamu.com
```
kemudian isi email dan `Agree Terms` Yes (Y) / No (N). Kemudian hasil cert akan tersimpan seperti berikut:

```
Certificate is saved at: /etc/letsencrypt/live/dns.znandev.net/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/dns.znandev.net/privkey.pem
```
Untuk menghindari error `reading cert file: open /etc/letsencrypt/live/dns.znandev.net/fullchain.pem: no such file or directory` kita harus edit `docker-compose.yml` dahulu karena AdGuard Home tidak memiliki akses langsung ke `/etc/letsencrypt` dihost:

Masuk kembali ke directory dihost:
```
cd /opt/adguardhome/
nano docker-compose.yml
```

Ubah bagian `volumes`:
```
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
      - /etc/letsencrypt:/etc/letsencrypt
```
Restart container:
```
sudo docker-compose down
sudo docker-compose up -d
```

#### Install SSL ke AdGuard Home

Buka `Settings` > `DNS Settings`.
Centang `Enable encryption (HTTPS, DNS-over-HTTPS and DNS-over-TLS)` kemudian isi:

 - Server Name: `dns.domainlu.com`
 - HTTPS port: `443`
 - DNS-over-TLS port: `853`
 - DNS-over-QUIC port: `784`
 - Lokasi/path untuk cert nya: 
    - `/etc/letsencrypt/live/dns.domainlu.com/fullchain.pem` 
    - `/etc/letsencrypt/live/dns.domainlu.com/privkey.pem`

Kemudian `Save configuration` dan notified `Encryption configuration saved` yaa.

Donee! Sekarang DNS Server kamu sudah terenkripsi sempurna.

Ada yang mau ditanyain? Komen aja dibawah ya.
