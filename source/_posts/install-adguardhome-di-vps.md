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
- Port : 80, 443, 53, 3000 (tcp dan udp)
2. Domain (opsional jika ingin gunakan DoH/DoT)

#### Persiapan

```apt update && apt upgrade -y
apt install -y docker.io docker-compose
systemctl enable docker
systemctl start docker
```

Cek

```
docker --version
docker-compose --version
```

#### ⚠️ Matikan DNS Default (Penting!)

Untuk hindari conflict port 53, matikan DNS Default. Cek apakah digunakan:

```
lsof -i :53
```

Jika benar digunakan silakan stop
```
systemctl stop systemd-resolved
systemctl disable systemd-resolved
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
mkdir -p /opt/adguardhome
cd /opt/adguardhome
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
      - "80:80/tcp"
      - "443:443/tcp"
      - "3000:3000/tcp"

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
http://IP-VPS:3000

> Note: Jangan lupa pastikan port 3000 tidak conflict


Ada yang mau ditanyain? Komen aja dibawah ya.
