# Command Reference — Homelab Infrastructure

Catatan pribadi: command yang pernah dieksekusi + penjelasan singkat, buat referensi cepat kalau ketemu masalah sama lagi atau mau rebuild dari nol.

---

## Directory Server — Samba AD DC (dc-server, 192.168.0.23)

### Persiapan VM & Sistem
```bash
ip a
```
Cek nama interface jaringan (misal `ens18`) sebelum setting IP statis.

```bash
sudo hostnamectl set-hostname dc-server
```
Set hostname VM.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```
Set IP statis (192.168.0.23) & DNS awal (192.168.0.20) via netplan.

```bash
ping -c 3 192.168.0.20
ping -c 3 8.8.8.8
```
Test konektivitas ke DNS internal & internet.

```bash
sudo apt update && sudo apt upgrade -y
```
Update sistem sebelum install apapun.

### Time Sync (wajib untuk Kerberos)
```bash
sudo apt install chrony -y
sudo systemctl status chrony
chronyc tracking
```
Install & cek NTP client — Kerberos gagal total kalau waktu antar server selisih >5 menit.

### Matikan konflik port 53
```bash
sudo ss -tlnp | grep :53
```
Cek apa yang sudah dengar di port 53 (biasanya `systemd-resolved`).

```bash
sudo nano /etc/systemd/resolved.conf
# set: DNSStubListener=no
sudo systemctl restart systemd-resolved
```
Matikan DNS stub listener bawaan Ubuntu, karena akan bentrok dengan Samba yang butuh port 53 sepenuhnya.

### Install paket Samba AD DC
```bash
sudo apt install samba krb5-config krb5-user winbind libnss-winbind libpam-winbind smbclient dnsutils -y
```
Install semua paket yang dibutuhkan. Saat instalasi `krb5-config`, isi realm dengan HURUF BESAR: `CORP.HOMELAB.INTERNAL`. Untuk KDC & administrative server, isi `dc-server.corp.homelab.internal`.

```bash
sudo systemctl status smbd
sudo systemctl status nmbd
```
Cek service Samba standalone yang otomatis jalan (normal sementara, akan digantikan `samba-ad-dc`).

### Provisioning jadi Domain Controller
```bash
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind
```
Matikan service standalone sebelum provisioning.

```bash
sudo rm /etc/samba/smb.conf
```
**Hanya jika muncul error** `'realm =' was not specified` saat provisioning — hapus config lama biar provisioning generate ulang dari nol.

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```
Command inti: mengubah Samba jadi Active Directory Domain Controller. Isian penting: Realm `CORP.HOMELAB.INTERNAL`, Domain `CORP`, Server Role `dc`, DNS backend `SAMBA_INTERNAL`, DNS forwarder `192.168.0.20`, plus password Administrator (harus kuat).

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```
Pasang file krb5.conf hasil generate provisioning ke lokasi sistem — wajib, jangan bikin symlink.

```bash
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
```
Aktifkan service AD DC yang sesungguhnya (menggantikan smbd/nmbd).

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
# ubah nameservers jadi: 127.0.0.1
sudo netplan apply
```
VM ini sekarang harus resolve dirinya sendiri untuk domain AD (karena DNS backend-nya `SAMBA_INTERNAL`).

### Verifikasi Domain Controller
```bash
host -t SRV _ldap._tcp.corp.homelab.internal
host -t SRV _kerberos._tcp.corp.homelab.internal
```
Cek SRV record inti AD (LDAP port 389, Kerberos port 88) sudah otomatis terbuat.

```bash
kinit administrator@CORP.HOMELAB.INTERNAL
klist
```
Test autentikasi Kerberos untuk akun Administrator, lalu cek tiket yang didapat.

### Delegasi DNS dari BIND9 (dikerjakan di dns-server, 192.168.0.20)
```bash
sudo nano /etc/bind/db.domain
```
Tambahkan NS + glue record (di zone `homelab.internal`):
```
corp                              IN  NS  dc-server.corp.homelab.internal.
dc-server.corp.homelab.internal.  IN  A   192.168.0.23
```
**Penting:** FQDN harus diakhiri titik (`.`), kalau tidak BIND anggap nama relatif dan glue record gagal terbaca.

```bash
sudo named-checkzone homelab.internal /etc/bind/db.domain
sudo systemctl reload bind9
```
Cek syntax & reload setelah naikkan serial number.

```bash
sudo nano /etc/bind/db.homelab
# tambah: 23   IN   PTR   dc-server.corp.homelab.internal.
```
Tambah PTR record untuk konsistensi reverse lookup.

```bash
sudo nano /etc/bind/named.conf.local
```
Tambahkan blok forward zone khusus (WAJIB, kalau tidak query akan selalu NXDOMAIN karena "dibajak" forwarder global 8.8.8.8):
```
zone "corp.homelab.internal" {
    type forward;
    forwarders { 192.168.0.23; };
};
```

```bash
sudo named-checkconf
sudo systemctl reload bind9
```
Cek syntax config utama & reload.

```bash
sudo rndc flush
```
**Kalau masih NXDOMAIN setelah semua fix di atas** — kemungkinan BIND9 cache jawaban negatif lama (bisa sampai 7 hari sesuai Negative Cache TTL). Flush paksa cache-nya.

### Debugging DNS Delegation (urutan cek kalau ada masalah)
```bash
dig @192.168.0.23 corp.homelab.internal
```
Test langsung ke Samba — pastikan Samba sendiri sehat & menjawab.

```bash
dig @192.168.0.20 corp.homelab.internal NS
```
Test ke BIND9 — kalau NXDOMAIN padahal Samba oke, masalah ada di sisi BIND9 (cache atau config forwarder).

```bash
dig +norecurse @192.168.0.20 corp.homelab.internal NS
```
Cek data zone LOKAL BIND9 tanpa lanjut resolve — kalau ini muncul benar (ada AUTHORITY + ADDITIONAL section berisi NS & glue record), berarti config zone file sudah benar, masalah ada di logic forwarding, bukan data.

```bash
nslookup dc-server.corp.homelab.internal 192.168.0.20
```
Test akhir dari perspektif client biasa — harus resolve ke 192.168.0.23.

### Bikin User & Grup
```bash
sudo samba-tool group add IT
sudo samba-tool group add Finance
```
Bikin grup baru (pengelompokan user berdasarkan divisi).

```bash
sudo samba-tool user create dara P@ssw0rd123!
```
Bikin user baru. Password harus kuat (kombinasi besar/kecil/angka/simbol) — AD menolak password lemah.

```bash
sudo samba-tool group addmembers IT dara
```
Masukkan user ke grup.

```bash
sudo samba-tool user list
```
Lihat semua user (termasuk akun bawaan: Administrator, Guest, krbtgt).

```bash
sudo samba-tool group listmembers IT
```
Lihat anggota grup tertentu.

```bash
kinit dara@CORP.HOMELAB.INTERNAL
```
Test login user biasa (bukan Administrator) untuk memastikan akun valid.

### Catatan Penting
- User AD (`dara@CORP.HOMELAB.INTERNAL`) **belum otomatis bisa dipakai** login ke mail-server/Roundcube — itu masih pakai user Linux lokal terpisah. Perlu domain join (winbind + PAM/NSS) di mail-server dulu untuk integrasi — belum dikerjakan.
- Domain AD (`corp.homelab.internal`) sengaja dipisah dari domain infra (`homelab.internal`) — karena Samba AD DC wajib jadi otoritas DNS penuh untuk domainnya sendiri, menyatukan keduanya akan bentrok dengan BIND9.

---

## Backup Server — Proxmox Backup Server (pbs-server, 192.168.0.25)

### Persiapan VM
```bash
# Provision VM baru pakai ISO Debian 13 (trixie) netinst — BUKAN Ubuntu
# IP statis 192.168.0.25, hostname pbs-server, nameserver ke 192.168.0.20
```
PBS resmi cuma didukung di Debian, dan versi PBS terbaru (4.x) berbasis Debian 13 (trixie), bukan lagi Debian 12 (bookworm).

### Install Proxmox Backup Server
```bash
echo "deb http://download.proxmox.com/debian/pbs trixie pbs-no-subscription" > /etc/apt/sources.list.d/pbs.list
wget https://enterprise.proxmox.com/debian/proxmox-release-trixie.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg
apt update
apt install proxmox-backup-server -y
```
Tambah repo resmi PBS (pakai codename `trixie` sesuai Debian 13), lalu install package-nya. Dijalankan sebagai root karena Debian minimal belum ada `sudo` ter-setup.

Saat instalasi, dialog Postfix (`postfix-configuration`) muncul karena PBS butuh kirim notifikasi email:
- **General type of mail configuration**: pilih `Internet with smarthost` — best practice-nya notifikasi backup/alerting dikirim independen dari infrastruktur yang dipantau (kalau numpang ke mail-server sendiri, pas mail-server down, notifikasi ikut nggak sampai).
- **System mail name**: `homelab.internal`
- **SMTP relay host**: `[smtp.gmail.com]:587` — tanda kurung siku wajib (maksa connect langsung ke host, bukan cari MX record), port 587 untuk STARTTLS.

### Setup notifikasi Gmail (SASL auth)
```bash
apt install libsasl2-modules -y
```
Modul tambahan yang dibutuhkan Postfix untuk autentikasi SASL ke SMTP luar (Gmail).

```bash
nano /etc/postfix/sasl_passwd
```
Isi 1 baris: `[smtp.gmail.com]:587    emailkamu@gmail.com:appPassword16digit`. App Password didapat dari [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords) — wajib aktifkan 2-Step Verification dulu di akun Google, Gmail tidak menerima password akun biasa untuk SMTP pihak ketiga.

```bash
postmap /etc/postfix/sasl_passwd
chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
```
`postmap` mengubah file teks jadi database yang bisa dibaca Postfix. `chmod 600` mengunci file berisi password mentah supaya cuma root yang bisa baca.

```bash
nano /etc/postfix/main.cf
```
Tambahkan di baris bawah:
```
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
header_size_limit = 4096000
```

```bash
systemctl restart postfix
echo "Test dari PBS" | mail -s "Test Email PBS" emailtujuan@gmail.com
```
Restart Postfix lalu kirim email test. **Catatan:** kalau muncul warning `overriding earlier entry: smtp_tls_security_level` — itu cuma info, bukan error fatal, karena Debian sudah punya baris default `smtp_tls_security_level=may` bawaan yang bentrok sama baris baru kita (`encrypt`). Postfix otomatis pakai yang paling akhir ditulis, email tetap terkirim normal. Bisa dibersihin nanti dengan `grep -n smtp_tls_security_level /etc/postfix/main.cf` untuk cari baris lama, lalu comment/hapus — tapi tidak mendesak.

### Konfigurasi PBS (web UI)
```
https://192.168.0.25:8007
```
Login pakai `root` + password Debian. Warning certificate self-signed itu normal, klik "Advanced > Proceed".

- **Datastore > Add Datastore** — nama `homelab-backup`, path penyimpanan (misal `/mnt/backup`, pastikan cukup ruang).
- Login ke Proxmox VE (`https://192.168.0.10:8006`) → **Datacenter > Storage > Add > Proxmox Backup Server** — isi Server `192.168.0.25`, Datastore `homelab-backup`, username `root@pam`, password root PBS.
- **Datacenter > Backup > Add** — pilih storage PBS, centang semua VM, atur jadwal (misal tiap jam 2 pagi), mode **Snapshot** (VM tetap jalan saat backup).

### Test backup & restore
```
Klik VM (misal dns-server) > Backup > Backup now, pilih storage PBS.
```
Test manual tanpa nunggu jadwal. Cek hasilnya muncul di datastore PBS.

```
Restore backup ke VM ID baru (misal 199, bukan ID asli).
```
**Step paling penting** — backup yang belum pernah dites restore-nya nggak bisa dipercaya. Kalau VM hasil restore boot normal dan semua service (termasuk resolusi DNS ke domain lain) masih jalan, berarti backup beneran valid. Hapus VM test setelah verifikasi selesai.

---

## VPN — WireGuard (Gagal) → Tailscale (Berhasil) (vpn-server, 192.168.0.26)

### Percobaan 1: WireGuard (tidak berhasil)

```bash
sudo apt install wireguard -y
```
Install WireGuard.

```bash
wg genkey | sudo tee /etc/wireguard/server_private.key
sudo cat /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key
```
Generate key pair server (autentikasi WireGuard pakai key, bukan password).

```bash
sudo nano /etc/wireguard/wg0.conf
```
Isi interface server: `Address = 10.10.10.1/24` (subnet baru khusus VPN), `ListenPort = 51820`, `PrivateKey`, plus `PostUp`/`PostDown` untuk iptables MASQUERADE (routing traffic VPN ke LAN).

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
Aktifkan IP forwarding — wajib supaya VPN server bisa nerusin paket ke LAN 192.168.0.x.

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg show
```
Aktifkan service. `wg show` menampilkan status interface + baris "latest handshake" (indikator utama koneksi client beneran jalan atau tidak).

**Setup pendukung (di luar VM):**
- Router D-Link DIR-612: menu **Virtual Server** (bukan "Port Forwarding") — isi Private IP `192.168.0.26`, Protocol **UDP**, port `51820`.
- Cek CGNAT dulu sebelum setup port forwarding: bandingkan WAN IP di halaman status router vs hasil search "what is my ip" — kalau beda, port forwarding tidak akan pernah berfungsi.
- DDNS via No-IP (karena DynDNS.com sudah berbayar, No-IP masih ada tier gratis): buat hostname baru di [my.noip.com](https://my.noip.com), **wajib centang "Enable Dynamic DNS"**, lalu isi hostname/username/password itu di menu DDNS router. Update period diisi `5` (menit).

```bash
wg genkey | tee client1_private.key | wg pubkey > client1_public.key
```
Generate key pair untuk client (HP), lalu didaftarkan sebagai `[Peer]` di `wg0.conf` server.

```bash
sudo apt install qrencode -y
qrencode -t ansiutf8 < client1.conf
```
Generate QR code dari file config client, di-scan langsung oleh app WireGuard di HP (tanpa transfer file manual).

**Diagnosis kegagalan (urutan troubleshooting):**
```bash
sudo wg show                                    # tidak ada "latest handshake" sama sekali
sudo tcpdump -i ens18 udp port 51820 -n         # 0 packets captured saat client coba connect
```
Dicoba ganti port ke `443` (kadang ISP kurang ketat di port umum) — tetap 0 packets. Test port TCP lain (8080) via canyouseeme.org **berhasil**, artinya port forwarding & jaringan rumah normal untuk TCP. Firewall Proxmox (VM/Node/Datacenter) dan UFW di VM semua sudah dicek nonaktif. **Kesimpulan:** ISP/operator seluler kemungkinan besar memblokir trafik UDP pada port non-standar — bukan salah konfigurasi, tapi pembatasan jaringan di luar kendali. WireGuard manual tidak bisa dipakai dalam kondisi ini.

### Percobaan 2: Tailscale (berhasil)

```bash
sudo systemctl stop wg-quick@wg0
sudo systemctl disable wg-quick@wg0
```
Matikan WireGuard yang gagal sebelum pindah solusi.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```
Install Tailscale — dibangun di atas protokol WireGuard yang sama, tapi punya NAT traversal otomatis (server relay DERP) sehingga tidak butuh port forwarding sama sekali.

```bash
sudo tailscale up --advertise-routes=192.168.0.0/24
```
Login via link yang muncul (autentikasi akun Google), lalu **advertise** subnet LAN — ini yang bikin device Tailscale lain bisa "menembus" ke jaringan 192.168.0.x lewat vpn-server sebagai gateway.

```
login.tailscale.com/admin/machines
```
Approve subnet route yang di-advertise tadi (device vpn-server > Edit subnet routes > centang `192.168.0.0/24` > Save) — wajib dilakukan manual sekali di web admin, tidak otomatis.

**Install Tailscale di HP/laptop**, login pakai akun yang sama — subnet routes otomatis ter-accept di Android/iOS.

```bash
sudo tailscale status
sudo tailscale ping <nama-device>
sysctl net.ipv4.ip_forward
```
Verifikasi status koneksi, test ping ke device lain, dan pastikan IP forwarding tetap aktif (masih wajib untuk subnet routing, sama seperti di WireGuard).

**Split DNS** (biar domain internal bisa di-resolve dari device remote):
```
login.tailscale.com/admin/dns → Add Nameserver
```
Nameserver `192.168.0.20`, aktifkan **Restrict to domain**, isi `homelab.internal` (dan tambah `familypet.com` sebagai domain kedua di nameserver yang sama).

### Catatan operasional
- `vpn-server` dan `dns-server` diset **Start at boot = Yes** di Proxmox — supaya remote access selalu bisa dilakukan kapan saja tanpa perlu nyalain manual.
- VM lain (web, mail, DC, file, PBS) dinyalakan manual sesuai kebutuhan lewat Proxmox web UI setelah terhubung VPN — menghemat resource RAM saat tidak dipakai.

---

## AD Domain Join — File Server (Samba Member via Winbind)

Integrasi `file-server` (Samba standalone) menjadi domain member dari `dc-server` (Samba AD DC), sehingga autentikasi share pakai akun & grup Active Directory (`CORP.HOMELAB.INTERNAL`), bukan lagi user/grup Linux lokal. Ini yang bikin dc-server akhirnya punya fungsi operasional nyata, bukan cuma bukti konsep berdiri sendiri.

### 1. Install Winbind (di file-server)
```bash
sudo apt install winbind libnss-winbind libpam-winbind -y
```
`libnss-winbind` otomatis menambahkan `winbind` ke `/etc/nsswitch.conf` (`passwd:` dan `group:` jadi include `winbind`). Verifikasi: `grep winbind /etc/nsswitch.conf`.

### 2. Konfigurasi domain member
```bash
sudo nano /etc/samba/smb.conf
```
Bagian `[global]`:
```ini
workgroup = CORP
realm = CORP.HOMELAB.INTERNAL
security = ads
idmap config * : backend = tdb
idmap config * : range = 3000-7999
idmap config CORP : backend = rid
idmap config CORP : range = 10000-999999
winbind use default domain = yes
```

### 3. Join domain
```bash
sudo net ads join -U administrator
```
Masukkan password Administrator AD. **Penting:** pastikan waktu (`date`) di file-server sinkron dengan dc-server sebelum join — Kerberos sensitif terhadap perbedaan waktu (sama seperti requirement chrony di awal setup dc-server).

```bash
sudo systemctl restart winbind smbd nmbd
```

### 4. Verifikasi domain join
```bash
wbinfo -u    # daftar user AD
wbinfo -g    # daftar grup AD
wbinfo -t    # cek trust secret ke domain controller
```

### 5. Bikin user & grup di AD (dijalankan di dc-server, BUKAN file-server)
```bash
sudo samba-tool user create <username> <password>
sudo samba-tool group addmembers <nama-grup> <username>
sudo samba-tool user show <username>
sudo samba-tool group listmembers <nama-grup>
```
**Catatan penting:** `samba-tool` cuma bisa dijalankan di server yang berperan sebagai Domain Controller (punya database `sam.ldb`) — tidak bisa dijalankan di domain member (file-server).

### 6. Update permission share (di file-server)
```bash
sudo nano /etc/samba/smb.conf
```
Ganti `valid users` dari grup Linux lokal jadi grup AD:
```ini
[IT]
   path = /srv/samba/it
   valid users = @"CORP\IT"
   read only = no
   browseable = yes

[Finance]
   path = /srv/samba/finance
   valid users = @"CORP\Finance"
   read only = no
   browseable = yes

[Public]
   path = /srv/samba/public
   valid users = @"CORP\Domain Users"
   read only = no
   browseable = yes
```

Cek GID grup AD, lalu ganti kepemilikan folder:
```bash
getent group "CORP\IT"
getent group "CORP\Finance"
getent group "CORP\Domain Users"

sudo chgrp "CORP\IT" /srv/samba/it
sudo chgrp "CORP\Finance" /srv/samba/finance
sudo chgrp "CORP\Domain Users" /srv/samba/public
```
Kalau `chgrp` pakai nama grup gagal, pakai GID numerik hasil `getent group` sebagai fallback: `chgrp <GID> <path>`.

```bash
sudo testparm
sudo systemctl restart smbd
```

### 7. Pengujian
Dari file-server (sisi Linux/Samba):
```bash
sudo apt install smbclient -y
smbclient //localhost/IT -U <username>
```

Dari Windows (Command Prompt):
```cmd
net use * /delete /y
net use Z: \\192.168.0.24\IT /user:CORP\<username>
```
Login berhasil dan folder bisa diakses kalau user itu anggota grup AD yang sesuai (`IT`, `Finance`, atau `Domain Users` untuk `Public`).

### Hasil akhir
- file-server berstatus domain member dari `CORP.HOMELAB.INTERNAL`
- Autentikasi share `IT`, `Finance`, `Public` sepenuhnya pakai akun & grup Active Directory
- Diuji dengan beberapa user AD, masing-masing cuma bisa akses share sesuai grupnya
- dc-server kini punya fungsi operasional nyata — bukan berdiri sendiri sebagai bukti konsep doang

### Belum dikerjakan
- Integrasi AD ke mail-server (Postfix/Dovecot) — lebih kompleks karena perlu ubah total cara autentikasi IMAP/SMTP, dijadikan proyek terpisah nanti.

---

## DNS Server (dns-server, 192.168.0.20)

### Menambah record baru
```bash
sudo nano /etc/bind/db.domain
```
Tambah A/CNAME/MX record sesuai kebutuhan. Contoh MX: `@ IN MX 10 mail.homelab.internal.`

Selalu naikkan serial number setelah edit, lalu:
```bash
sudo named-checkzone homelab.internal /etc/bind/db.domain
sudo systemctl reload bind9
```

### PTR record (reverse zone)
```bash
sudo nano /etc/bind/db.homelab
```
Tambah PTR sesuai IP baru. Best practice: 1 PTR per IP — jangan biarkan 1 IP punya beberapa PTR ke hostname berbeda.

### Kesalahan umum & fix
- **Karakter nyasar di baris pertama** (misal `0;` sebelum baris komentar) bikin BIND gagal parsing total sejak baris 1 → hapus baris itu.
- **FQDN tanpa titik di akhir** dianggap nama relatif oleh BIND, otomatis ditambahin origin zone di belakangnya → selalu akhiri FQDN dengan titik (`.`), terutama di glue record.
- **Baris tanpa nama di kolom depan** otomatis "nempel" ke owner name baris sebelumnya (misal buat nulis MX/PTR tambahan) — valid secara syntax, tapi bisa bikin bingung baca ulang. Indentasi spasi/tab di depan itu yang jadi sinyal "nyambung ke baris atas".

### Windows tidak resolve domain internal
Penyebab umum: Windows pakai DNS server lain (kadang resolver IPv6 link-local dari router), bukan `192.168.0.20`. Fix: set DNS manual di adapter Windows ke `192.168.0.20`; kalau masih gagal, matikan IPv6 di adapter itu dan `ipconfig /flushdns`.

### Tambah zone/domain baru (contoh: familypet.com)
```bash
sudo nano /etc/bind/named.conf.local
```
Tambah blok baru di bawah zone yang sudah ada:
```
zone "familypet.com" {
    type master;
    file "/etc/bind/db.familypet";
};
```
```bash
sudo cp /etc/bind/db.domain /etc/bind/db.familypet
sudo nano /etc/bind/db.familypet
```
Sesuaikan semua nama domain di dalamnya, reset serial ke `1`, hapus record yang tidak relevan (mail/web dari domain lama).
```bash
sudo named-checkzone familypet.com /etc/bind/db.familypet
sudo systemctl reload bind9
```

### Delegasi DNS ke domain lain (misal ke Samba AD DC)
Lihat detail lengkap di bagian **Directory Server** di bawah — mencakup NS + glue record, forward zone khusus (supaya tidak dibajak forwarder global), dan `rndc flush` untuk cache negatif.

---

## Web Server (web-server, 192.168.0.21)

### Provisioning & Nginx dasar
```bash
sudo hostnamectl set-hostname web-server
sudo nano /etc/netplan/50-cloud-init.yaml   # IP statis, gateway, nameserver ke 192.168.0.20
sudo netplan apply
sudo apt update && sudo apt upgrade -y
sudo apt install nginx -y
```

### PHP-FPM
```bash
apt-cache policy php-fpm     # cek versi PHP yang akan terinstall
sudo apt install php-fpm php-mysql -y
```
Aktifkan pemrosesan PHP di server block Nginx: uncomment blok `location ~ \.php$ { ... }`, arahkan `fastcgi_pass` ke socket versi PHP yang benar (misal `unix:/run/php/php8.3-fpm.sock`).

### Multi-site (banyak domain, 1 Nginx)
Setiap domain punya file config sendiri di `sites-available/`, diaktifkan lewat symlink:
```bash
sudo ln -s /etc/nginx/sites-available/<nama> /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```
**Cuma boleh 1 file** yang punya `listen 80 default_server;` di seluruh `sites-enabled/` — duplikat bikin error "duplicate default server".

### Dummy default_server (security hardening)
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}
```
Menolak akses langsung via IP tanpa menampilkan situs manapun.

### SSL/HTTPS self-signed
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/homelab/<domain>.key \
  -out /etc/ssl/homelab/<domain>.crt \
  -subj "/CN=<domain>"
```
Tambah server block port 443 (`listen 443 ssl;`, `ssl_certificate`, `ssl_certificate_key`), lalu ubah server block port 80 yang lama jadi redirect: `return 301 https://$host$request_uri;`

### Security headers
Di `/etc/nginx/nginx.conf` (blok `http {}`): `server_tokens off;`
Di tiap server block HTTPS: `add_header X-Frame-Options "SAMEORIGIN";` dan `add_header X-Content-Type-Options "nosniff";`

### Koneksi PHP ke MySQL/MariaDB
Error `Class "mysqli" not found` → ekstensi belum terinstall: `sudo apt install php-mysql -y`, restart PHP-FPM.

### SSH host key warning setelah reinstall OS
```powershell
ssh-keygen -R <IP>
```
Hapus entry lama di `known_hosts` Windows — normal terjadi tiap kali OS di-install ulang di IP yang sama.

---

## Mail Server (mail-server, 192.168.0.22)

### Instalasi
```bash
sudo apt install postfix dovecot-imapd dovecot-pop3d -y
```
Saat instalasi: pilih **Internet Site**, isi System mail name dengan domain EMAIL (`homelab.internal`) — BUKAN hostname server (`mail.homelab.internal`).

### Konfigurasi Postfix inti
```bash
sudo nano /etc/postfix/main.cf
```
Tambah di baris akhir: `home_mailbox = Maildir/` (format penyimpanan mailbox). `mynetworks` dibatasi internal-only, TIDAK pakai `0.0.0.0/0` (yang membuka relay ke seluruh internet).

### Konfigurasi Dovecot
```bash
sudo nano /etc/dovecot/conf.d/10-mail.conf
```
`mail_location = maildir:~/Maildir` — harus cocok dengan `home_mailbox` di Postfix.

### DNS resolver mail-server sendiri
Mail-server WAJIB bisa resolve domain internal (nameserver diarahkan ke `192.168.0.20` di netplan) — tanpa ini, Roundcube gagal konek ke Dovecot dengan error `getaddrinfo failed`.

### Perbedaan hostname vs domain email
- `mail.homelab.internal` → hostname server (dipakai di DNS/MX record)
- `homelab.internal` → domain email (dipakai di alamat `user@homelab.internal`)

Alamat email harus pakai domain (sesuai `mydestination` di Postfix), BUKAN hostname — kirim ke `user@mail.homelab.internal` akan gagal karena domain itu tidak terdaftar di `mydestination`.

### Roundcube + MariaDB
```bash
sudo apt install roundcube roundcube-mysql -y
```
Kalau muncul error `Can't connect to local server through socket` saat setup database → server database (MariaDB) belum terinstall:
```bash
sudo apt install mariadb-server -y
sudo systemctl start mariadb
sudo mysql_secure_installation
sudo dpkg-reconfigure roundcube-core
```

### Konfigurasi Roundcube
```bash
sudo nano /etc/roundcube/config.inc.php
```
```php
$config['imap_host'] = ["mail.homelab.internal:143"];
$config['smtp_host'] = 'mail.homelab.internal:25';
$config['mail_domain'] = 'homelab.internal';   // wajib, biar identity user otomatis pakai domain email, bukan hostname
```

### SSL untuk Dovecot & Roundcube
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/dovecot.key \
  -out /etc/ssl/certs/dovecot.crt \
  -subj "/CN=mail.homelab.internal"
```
Di `/etc/dovecot/conf.d/10-ssl.conf`: `ssl = yes`, arahkan `ssl_cert`/`ssl_key` ke file di atas. Setelah SSL aktif, kembalikan `disable_plaintext_auth` di `10-auth.conf` dari `no` ke `yes`.

Update Roundcube ke IMAP SSL (port 993):
```php
$config['imap_host'] = ["ssl://mail.homelab.internal:993"];
```
Kalau muncul error `SSL_accept() failed: unknown ca` → PHP menolak verifikasi cert self-signed, tambahkan:
```php
$config['imap_conn_options'] = [
    'ssl' => [
        'verify_peer'       => false,
        'verify_peer_name'  => false,
        'allow_self_signed' => true,
    ],
];
```

Untuk HTTPS di Roundcube sendiri (Apache): aktifkan `mod_ssl` (`a2enmod ssl`), buat virtual host port 443 pakai certificate yang sama (`dovecot.crt`/`dovecot.key`), karena domainnya sama (`mail.homelab.internal`).

---

*(Command reference lengkap: DNS, Web, Mail, Directory Server, Backup Server, AD Domain Join, VPN)*
