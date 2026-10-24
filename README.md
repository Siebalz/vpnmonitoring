# Pantau Jaringan — monitoring MikroTik + manajemen VPN client (satu web)

Web monitoring self-hosted untuk puluhan MikroTik yang terhubung ke server VPN Anda.
Server ini menarik data lewat **RouterOS API** ke IP VPN tiap router (tanpa agent/script di router).

**Fitur**
- Dashboard: status semua router, total traffic uplink, alert aktif, filter grup/status/cari
- Detail per router: traffic uplink, CPU, RAM, latency, suhu, klien DHCP/WiFi/VPN, conntrack, ketersediaan (strip uptime), traffic per interface, info perangkat
- Rentang grafik 1 jam – 30 hari, crosshair + tooltip, garis batas alert, tema gelap/terang
- Alert Telegram: router **DOWN/pulih**, CPU, RAM, suhu, latency tinggi, **interface down** (interface yang diberi bintang), **reboot**
- Bot Telegram interaktif: `/status`, `/routers`, `/router <nama>`, `/alerts`, `/id`
- Password router disimpan terenkripsi (Fernet); login admin; semua data di SQLite (`data/monitor.db`)

## Fitur VPN (digabung dari VPN Manager PHP)

Menu **VPN** mengelola server MikroTik L2TP Anda lewat RouterOS API:

- Tambah/hapus user VPN: PPP secret + **IP statis otomatis** + **port-forward (NAT) otomatis** (Winbox, API, WebFig, HTTPS — daftar & rentang port bisa diatur di Pengaturan)
- Daftar user dengan status online, uptime, traffic RX/TX, dan alamat akses publik
- **Script client** siap tempel untuk router pelanggan (L2TP client), bisa diambil ulang kapan saja
- Nonaktifkan/aktifkan user (sesi aktif ikut diputus), pembersihan NAT yatim
- **Terintegrasi dengan monitoring**: centang *Daftarkan ke monitoring* saat membuat user → router otomatis masuk daftar pantau (host = IP VPN-nya) dan script client ikut membuat user read-only `monitor` + mengaktifkan API. Hapus user VPN = router ikut hilang dari monitoring.
- Semua aksi VPN tercatat di **Riwayat alert** (level Info, diawali nama admin) — pengganti Log Aktivitas lama.

Langkah awal: menu **VPN → Tambah server VPN**, isi host API + akun admin MikroTik server VPN, IP publik, dan subnet VPN (mis. `11.11.11.`), lalu **Tes koneksi**.

> Akun MikroTik untuk server VPN butuh hak tulis (`ppp`, `api`, `write`, `policy`, `sensitive` agar password secret bisa dibaca untuk script ulang). Pakai akun terpisah dari akun read-only monitoring.
>
> Alamat `address=` pada user `monitor` di script client default-nya subnet VPN /24. Bila server Pantau ini mengakses router client dari IP lain (mis. lewat NAT/masquerade), isi **Pengaturan → Sumber akses API monitoring**.

### Migrasi dari aplikasi PHP lama
```bash
python tools/import_vpnserver.py /path/ke/vpnserver/data   # butuh mikrotiks.json + .enckey
```
Router terdaftar di PHP diimpor sebagai server VPN (password didekripsi lalu dienkripsi ulang dengan kunci aplikasi ini). Setelah itu folder PHP boleh dimatikan.

### Tes
```bash
pip install -r requirements.txt httpx
python tests/test_vpn.py     # memakai RouterOS palsu, tanpa router sungguhan
```

## 1. Siapkan tiap MikroTik (sekali per router)

Buat user **read-only** dan batasi API hanya dari subnet VPN (ganti `10.8.0.0/24` dengan subnet VPN Anda):

```
/user group add name=monitor policy=read,api
/user add name=monitor group=monitor password=GANTI_PASSWORD_KUAT address=10.8.0.0/24
/ip service set api disabled=no address=10.8.0.0/24
/ip service set api-ssl disabled=yes
```
Bila firewall `input` Anda memblokir, izinkan `tcp/8728` dari subnet VPN. Panduan ini juga ada di menu **Router → Panduan setup MikroTik**.

> Server monitoring harus bisa **ping/koneksi ke IP VPN router**. Paling mudah: pasang aplikasi ini **di server VPN itu sendiri** (atau server yang terhubung ke VPN yang sama).

## 2. Jalankan

**Docker (disarankan)**
```bash
cp .env.example .env      # edit ADMIN_PASSWORD
docker compose up -d --build
```
**Tanpa Docker**
```bash
cp .env.example .env      # edit ADMIN_PASSWORD
./run.sh
```
Buka `http://IP-SERVER:8080`, login dengan `ADMIN_USER` / `ADMIN_PASSWORD`.

Ingin lihat tampilannya dulu? Set `DEMO=1` di `.env` (8 router simulasi + riwayat 24 jam), hapus folder `data/` dan set `DEMO=0` setelah selesai mencoba.

## 3. Tambah router
Menu **Router → Tambah router**: isi nama, IP VPN, username/password user `monitor`, lalu **Tes koneksi**
(akan menampilkan model & versi RouterOS, dan daftar interface — klik salah satu untuk dijadikan *uplink* grafik utama).
Mengosongkan uplink = otomatis (interface fisik dengan traffic terbesar).

## 4. Telegram
1. Chat **@BotFather** → `/newbot` → salin token.
2. **Pengaturan → Notifikasi Telegram**: tempel token, simpan.
3. Kirim `/id` ke bot (atau tambahkan bot ke grup lalu kirim `/id`) → bot membalas chat ID. Isi chat ID (boleh banyak, pisahkan koma).
4. Klik **Kirim pesan tes**.

Server hanya membutuhkan koneksi **keluar** ke `api.telegram.org` (bot memakai long-polling, tidak perlu domain/webhook).

## Cara kerja alert
| Alert | Kondisi |
|---|---|
| Router DOWN | gagal poll `offline_after` kali berturut-turut (default 3 × 30 dtk ≈ 90 dtk) |
| CPU / RAM / Suhu / Latency | di atas batas selama `sustain` poll berturut-turut (default 3); pulih memakai histeresis agar tidak flapping |
| Interface down | hanya interface yang diberi ⭐ di halaman detail router |
| Reboot | uptime router tiba-tiba mereset |

Batas bisa diatur global (Pengaturan) dan di-override per router. Notifikasi pulih memuat lama gangguan. Tiap alert tidak dikirim berulang kecuali "Ulangi alert aktif" diisi.

## Keamanan
- Ganti `ADMIN_PASSWORD`. Pasang di belakang HTTPS (Caddy/Nginx/Traefik) bila diakses lewat internet, atau akses hanya lewat VPN.
- Pakai user MikroTik read-only dan batasi `address=` ke subnet VPN.
- Cadangkan folder `data/` (berisi database dan `secret.key` untuk dekripsi password router).

## Kapasitas & penyimpanan
Polling 30 dtk: ±2.900 baris/router/hari untuk metrik (disimpan 60 hari) dan ±8 interface × 2.900 untuk traffic interface (7 hari). 50 router ≈ beberapa ratus MB. Atur di **Pengaturan → Polling & penyimpanan**.

## Struktur
```
app/ros.py       klien RouterOS API (+ simulator DEMO)
app/poller.py    polling paralel, hitung bps, mesin alert
app/vpn.py       user VPN: PPP secret, NAT, script client
app/telegram.py  kirim notifikasi + bot perintah
app/main.py      REST API + login
app/static/      dashboard (Chart.js & font dibundel, jalan offline)
```
Ide pengembangan: impor otomatis router dari `/ppp secret` server VPN MikroTik, SNMP untuk perangkat non-MikroTik, ringkasan harian ke Telegram.
# webmonitoring
# vpnmonitoring
