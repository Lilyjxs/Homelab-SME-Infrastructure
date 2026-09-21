# Command Reference

command yang pernah dieksekusi, serta penjelasan singkat buat referensi kalau ketemu masalah sama lagi atau mau rebuild dari nol.

---

## DNS Server (dns-server, 192.168.0.20)

### Isi akhir zone file
```
; /etc/bind/db.domain => Isi file forward zone homelab.internal
@       IN      NS      homelab.internal.
@       IN      A       192.168.0.20
@       IN      MX      10      mail.homelab.internal.
ns      IN      CNAME   @
www     IN      CNAME   @
demo    IN      A       192.168.0.21
web     IN      A       192.168.0.21
mail    IN      A       192.168.0.22
corp    IN      NS      dc-server.corp.homelab.internal.
dc-server.corp.homelab.internal.  IN  A  192.168.0.23
```
```
; /etc/bind/db.homelab => Isi file reverse zone 0.168.192.in-addr.arpa
20      IN      PTR     ns.homelab.internal.
21      IN      PTR     web.homelab.internal.
22      IN      PTR     mail.homelab.internal.
23      IN      PTR     dc-server.corp.homelab.internal.
```

### Alur nambah record baru (pola yang berulang tiap kali ada service baru)
```bash
sudo nano /etc/bind/db.domain
```
Tambah baris A record baru (misal `demo IN A 192.168.0.21` saat web-server mulai host situs demo). Naikkan angka Serial di blok SOA.
```bash
sudo named-checkzone homelab.internal /etc/bind/db.domain
sudo systemctl reload bind9
```
Verifikasi tiap domain satu-satu setelah reload:
```bash
nslookup homelab.internal
nslookup web.homelab.internal
nslookup demo.homelab.internal
nslookup mail.homelab.internal
```

### MX & PTR record
```bash
sudo nano /etc/bind/db.domain
# tambahin: @  IN  MX  10  mail.homelab.internal.
sudo nano /etc/bind/db.homelab
# tambahin: 22  IN  PTR  mail.homelab.internal.
```
Verifikasi:
```bash
nslookup NamaDomain
nslookup -type=PTR IPNameServer
```

### Kesalahan umum & fix
- **Baris tanpa nama di kolom depan** indentasi spasi/tab otomatis "nempel" ke owner name baris sebelumnya bakal valid secara syntax buat nulis beberapa record dengan owner sama (misal 3 PTR untuk 1 IP), tapi jadi bug ketika tidak disengaja.
- **PTR duplikat untuk 1 IP** (misal `.20` sempat punya 3 PTR: ke `homelab.internal`, `ns.homelab.internal`, `www.homelab.internal`) dirapihin jadi 1 PTR per IP sesuai konvensi standar (`ns.homelab.internal`).

### Windows tidak resolve domain internal (urutan diagnosis)
```
nslookup homelab.internal          # dari Windows, cek Server yang dipakai
```
Kalau muncul `Server: UnKnown` dengan `Address` berupa alamat IPv6 link-local (`fe80::...`) bisa jadi itu tandanya Windows pakai resolver lain, bukan `192.168.0.20`. Fix bertahap:
1. Set DNS manual di Settings > Network > adapter aktif > ganti ke Manual, isi Preferred DNS `192.168.0.20`.
2. Kalau masih gagal, matikan IPv6 di adapter yang sama (banyak kasus Windows tetap prioritaskan resolver IPv6 walau DNS IPv4 sudah diset manual).
3. `ipconfig /flushdns` lalu test ulang `nslookup homelab.internal`.
4. Cek firewall di dns-server tidak memblokir port 53: `sudo ufw status` (kalau aktif dan belum ada rule, `sudo ufw allow 53`).

### Tambah zone/domain baru, sepenuhnya terpisah (contoh: familypet.com)
```bash
sudo nano /etc/bind/named.conf.local
```
Tambah blok baru di bawah zone yang sudah ada dibawahnya jika mau nambah domain:
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
Isi akhir (disederhanakan, tidak perlu record mail/web dari domain lama):
```
$TTL    604800
@       IN      SOA     familypet.com. root.familypet.com. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
@       IN      NS      familypet.com.
@       IN      A       192.168.0.21
www     IN      A       192.168.0.21
```
```bash
sudo named-checkzone familypet.com /etc/bind/db.familypet
sudo systemctl reload bind9
nslookup familypet.com
```

### Delegasi DNS ke domain lain (misal ke Samba AD DC)
Lihat detail lengkap di bagian **Directory Server** yang mana mencakup NS + glue record, forward zone khusus (supaya tidak dibajak forwarder global), dan `rndc flush` untuk cache negatif.

---

## Web Server (web-server, 192.168.0.21)

### Provisioning dari Ubuntu kosongan
```bash
ip a
sudo hostnamectl set-hostname web-server
sudo nano /etc/netplan/50-cloud-init.yaml
```
Isi netplan:
```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses: [192.168.0.21/24]
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses: [192.168.0.20]
```
```bash
sudo netplan apply
sudo apt update && sudo apt upgrade -y
sudo apt install nginx -y
sudo systemctl status nginx
```

### Install PHP-FPM
```bash
# cek versi
apt-cache policy php-fpm
sudo apt install php-fpm php-mysql -y
# cek nama file socket semisal php8.3-fpm.sock
sudo ls -l /run/php/                          
```
Backup config sebelum diedit:
```bash
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak
```
Aktifin pemrosesan PHP di file server block, tambah `index.php` di baris `index`, dan uncomment blok:
```nginx
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
}
```
Test dengan file `phpinfo()`, lalu **wajib dihapus** setelah tes berhasil (data sensitif kalau kebuka publik):
```bash
sudo nano /var/www/html/info.php    
# isi: <?php phpinfo(); ?>

# test di browser: http://web.homelab.internal/info.php
sudo rm /var/www/html/info.php
```

### Multi-site, struktur akhir 3 file terpisah
Setiap domain punya file config sendiri di `sites-available/`, diaktifkan lewat symlink ke `sites-enabled/`:
```bash
sudo ln -s /etc/nginx/sites-available/<nama> /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

**File `catchall`** buat dummy default_server, menolak akses via IP langsung:
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}
```

**File `demo`** buat website demo:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name demo.homelab.internal;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name demo.homelab.internal;
    root /var/www/demo;
    index index.php index.html;
    ssl_certificate     /etc/ssl/homelab/demo.homelab.internal.crt;
    ssl_certificate_key /etc/ssl/homelab/demo.homelab.internal.key;
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
}
```

**File `familypet`** domain utama:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name familypet.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name familypet.com;
    root /var/www/html;
    index index.php index.html;
    ssl_certificate     /etc/ssl/homelab/familypet.com.crt;
    ssl_certificate_key /etc/ssl/homelab/familypet.com.key;
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
}
```
**Cuma boleh 1 file** (`catchall`) yang punya flag `default_server` kalo duplikat di file lain bikin error `a duplicate default server for 0.0.0.0:80`.

### SSL/HTTPS self-signed (per domain)
```bash
sudo mkdir -p /etc/ssl/homelab
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/homelab/<domain>.key \
  -out /etc/ssl/homelab/<domain>.crt \
  -subj "/CN=<domain>"
```
Diulang untuk tiap domain (`demo.homelab.internal`, `familypet.com`, dll).

### Security headers
Di `/etc/nginx/nginx.conf` (dalam blok `http {}`):
```
server_tokens off;
```
Di tiap server block HTTPS:
```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
```
Verifikasi: `curl -I https://familypet.com -k` cek tidak ada lagi baris `Server: nginx/1.24.0`, dan ada baris `X-Frame-Options`.

### Koneksi PHP ke MySQL/MariaDB
```bash
sudo apt install mariadb-server -y     
sudo apt install php-mysql -y
sudo systemctl restart php8.3-fpm
# verifikasi ekstensi aktif
php -m | grep mysqli                   
```
Error `Class "mysqli" not found` → ekstensi `php-mysql` belum terinstall, bukan masalah kode PHP-nya.

### SSH host key warning setelah reinstall OS
```powershell
ssh-keygen -R <IP>
```
Hapus entry lama di `known_hosts` Windows, normal terjadi tiap kali OS di-install ulang di IP yang sama (host key baru berbeda dari sebelumnya).

---

## Mail Server (mail-server, 192.168.0.22)

### Provisioning dari Ubuntu kosongan
```bash
sudo hostnamectl set-hostname mail-server
# pake IP statis, nameserver ke DNS UTAMA
sudo nano /etc/netplan/50-cloud-init.yaml   
sudo netplan apply
sudo apt update && sudo apt upgrade -y
```

### Instalasi Postfix + Dovecot
```bash
sudo apt install postfix dovecot-imapd dovecot-pop3d -y
```
Dialog konfigurasi Postfix (debconf) yang muncul:
- **General type of mail configuration**: `Internet Site`
- **System mail name**: `homelab.internal`(samain sama domain utama) domain EMAIL, BUKAN hostname server (`mail.homelab.internal`); salah isi di sini bikin alamat email jadi `user@mail.homelab.internal` yang tidak sesuai MX record.
- **Local networks**: `127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 192.168.0.0/24` sengaja TIDAK diisi `0.0.0.0/0` karena itu membuka relay ke seluruh internet (open relay risk).
- **Mailbox size limit**: `0` (unlimited)
- **Local address extension character**: `+` (default, fitur `user+tag@domain` ala Gmail)

### Konfigurasi Postfix tambahan
```bash
sudo nano /etc/postfix/main.cf
```
Tambah di baris akhir:
```
home_mailbox = Maildir/
```
Format penyimpanan mailbox (Maildir, bukan Mbox) wajib ada karena default Postfix ga set ini.

```bash
sudo systemctl restart postfix
sudo systemctl status postfix
```

### Konfigurasi Dovecot
```bash
sudo nano /etc/dovecot/conf.d/10-mail.conf
# ubah: mail_location = maildir:~/Maildir
```
Harus cocok dengan `home_mailbox` di Postfix.

```bash
sudo nano /etc/dovecot/conf.d/10-auth.conf
# awal: disable_plaintext_auth = no  (sementara, karena belum ada SSL)
```

```bash
sudo systemctl restart dovecot
```

### Bikin user testing
```bash
sudo adduser NamaUser
```
Tiap user otomatis dapat mailbox lewat `home_mailbox`/`mail_location` yang sudah diset.

### DNS resolver mail-server sendiri
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
# pastikan nameservers: [192.168.0.20]
sudo netplan apply
nslookup mail.homelab.internal
```
Mail-server WAJIB bisa resolve domain internal, kalo ini ga bisa roundcube gagal konek ke Dovecot dengan error `getaddrinfo failed: Name or service not known`.

### Perbedaan hostname vs domain email
- `mail.homelab.internal`: hostname server (dipakai di DNS A/MX record)
- `homelab.internal`: domain email (dipakai di alamat `user@homelab.internal`)

Alamat email harus pakai domain sesuai `mydestination` di Postfix, kirim ke `user@mail.homelab.internal` gagal karena domain itu tidak terdaftar sebagai tujuan lokal, hanya `$myhostname` dan `homelab.internal` yang dikenali.

### Roundcube + MariaDB
```bash
sudo apt install roundcube roundcube-mysql -y
```
Saat instalasi minta password root database untuk dbconfig-common, kalau muncul error:
```
ERROR 2002 (HY000): Can't connect to local server through socket '/run/mysqld/mysqld.sock'
```
berarti server database belum terinstall sama sekali (baru ekstensi client saja yang ter-install dari dependency Roundcube):
```bash
sudo apt install mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
sudo dpkg-reconfigure roundcube-core
```

### Konfigurasi Roundcube
```bash
sudo nano /etc/roundcube/config.inc.php
```
```php
// atau mail.homelab.internal:143 jika DNS resolver sudah beres
$config['imap_host'] = ["localhost:143"];
$config['smtp_host'] = 'localhost:25';
// wajib, agar identity user baru otomatis @homelab.internal, bukan ikut domain imap_host
$config['mail_domain'] = 'homelab.internal';  
```
Semisal user yang identity-nya salah **Settings > Identities** di Roundcube per-user.

### SSL untuk Dovecot
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/dovecot.key \
  -out /etc/ssl/certs/dovecot.crt \
  -subj "/CN=mail.homelab.internal"
```
```bash
sudo nano /etc/dovecot/conf.d/10-ssl.conf
```
```
ssl = yes
ssl_cert = </etc/ssl/certs/dovecot.crt
ssl_key = </etc/ssl/private/dovecot.key
```
Tanda `<` di depan path itu wajib, bagian dari syntax Dovecot.
```bash
sudo nano /etc/dovecot/conf.d/10-auth.conf
# ubah command: disable_plaintext_auth = yes
sudo systemctl restart dovecot
```

Update Roundcube ke IMAP SSL (port 993):
```php
$config['imap_host'] = ["ssl://mail.homelab.internal:993"];
```
Error `SSL_accept() failed: unknown ca` PHP mnolak verifikasi certificate self-signed:
```php
$config['imap_conn_options'] = [
    'ssl' => [
        'verify_peer'       => false,
        'verify_peer_name'  => false,
        'allow_self_signed' => true,
    ],
];
```

### HTTPS untuk Roundcube sendiri (Apache)
```bash
sudo a2enmod ssl
sudo nano /etc/apache2/sites-available/mail-ssl.conf
```
```apache
<VirtualHost *:443>
    ServerName mail.homelab.internal
    DocumentRoot /var/lib/roundcube
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/dovecot.crt
    SSLCertificateKeyFile /etc/ssl/private/dovecot.key
    <Directory /var/lib/roundcube>
        Options +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```
Certificate yang sama dipakai ulang dan bukan bikin baru karena domainnya sama persis (`mail.homelab.internal`).
```bash
sudo a2ensite mail-ssl.conf
sudo apache2ctl configtest
sudo systemctl restart apache2
```

### Verifikasi akhir
```bash
echo "test" | mail -s "Test" NamaUser@homelab.internal
tail -f /var/log/mail.log
```
Login Roundcube via `https://mail.homelab.internal`, kirim & terima email antar-user internal (`@homelab.internal`, bukan `@mail.homelab.internal`) berhasil dua arah.

---

## Directory Server, Samba AD DC (dc-server, 192.168.0.23)

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
Install & cek NTP client, Kerberos bakal gagal total kalau waktu antar server selisih >5 menit.

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
**Hanya jika muncul error** `'realm =' was not specified` saat provisioning hapus file config lama biar provisioning generate ulang dari nol.

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```
Command inti: mengubah Samba jadi Active Directory Domain Controller. Isian penting: Realm `CORP.HOMELAB.INTERNAL`, Domain `CORP`, Server Role `dc`, DNS backend `SAMBA_INTERNAL`, DNS forwarder `192.168.0.20`, plus password Administrator.

```bash
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```
Pasang file krb5.conf hasil generate provisioning ke lokasi sistem sifatnya wajib, jangan bikin symlink.

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

### Delegasi DNS dari BIND9 (dikerjakan di dns-server)
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
**Kalau masih NXDOMAIN setelah semua fix di atas** kemungkinan BIND9 cache jawaban negatif lama (bisa sampai 7 hari sesuai Negative Cache TTL). bersihin dengan Flush cache-nya.

### Debugging DNS Delegation (urutan cek kalau ada masalah)
```bash
dig @192.168.0.23 corp.homelab.internal
```
Test langsung ke Samba

```bash
dig @192.168.0.20 corp.homelab.internal NS
```
Test ke BIND9, kalau NXDOMAIN padahal Samba oke, masalah ada di sisi BIND9, bisa di cache atau config forwarder.

```bash
dig +norecurse @192.168.0.20 corp.homelab.internal NS
```
Cek data zone LOKAL BIND9 tanpa lanjut resolve, kalau ini muncul benar (ada AUTHORITY + ADDITIONAL section berisi NS & glue record), berarti config zone file sudah benar, masalah ada di logic forwarding, bukan data.

```bash
nslookup dc-server.corp.homelab.internal 192.168.0.20
```

### Bikin User & Grup
```bash
sudo samba-tool group add IT
sudo samba-tool group add Finance
```
Bikin grup baru buat mengelompokan user berdasarkan divisi

```bash
sudo samba-tool user create NamaUser IsiPassword
```
Bikin user baru. Password harus kuat, AD menolak password lemah.

```bash
sudo samba-tool group addmembers IT NamaUser
```
Masukkan user ke grup.

```bash
sudo samba-tool user list
```
Lihat semua user

```bash
sudo samba-tool group listmembers IT
```
Lihat anggota grup tertentu.

```bash
kinit NamaUser@CORP.HOMELAB.INTERNAL
```
Test login user biasa (bukan Administrator) untuk memastikan akun valid.

### Catatan Penting
- User AD (`dara@CORP.HOMELAB.INTERNAL`) **belum otomatis bisa dipakai** login ke mail-server/Roundcube masih pakai user lokal terpisah. Perlu domain join (winbind + PAM/NSS) di mail-server dulu untuk integrasi.
- Domain AD (`corp.homelab.internal`) sengaja dipisah dari domain infra (`homelab.internal`) karena Samba AD DC wajib jadi otoritas DNS penuh untuk domainnya sendiri, menyatukan keduanya akan bentrok dengan BIND9.

---

## File Server, AD Domain Join (file-server, 192.168.0.24)

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
Masukkan password Administrator AD. **Penting:** pastikan waktu (`date`) di file-server sinkron dengan dc-server sebelum join Kerberos sensitif terhadap perbedaan waktu.

```bash
sudo systemctl restart winbind smbd nmbd
```

### 4. Verifikasi domain join
```bash
# daftar user AD
wbinfo -u    
# daftar grup AD
wbinfo -g    
# cek trust secret ke domain controller
wbinfo -t   
```

### 5. Bikin user & grup di AD (dijalankan di dc-server, BUKAN file-server)
```bash
sudo samba-tool user create <username> <password>
sudo samba-tool group addmembers <nama-grup> <username>
sudo samba-tool user show <username>
sudo samba-tool group listmembers <nama-grup>
```
**Catatan penting:** `samba-tool` cuma bisa dijalankan di server yang berperan sebagai Domain Controller yang punya database `sam.ldb` ia tidak bisa dijalankan di domain member yaitu file-server.

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
- dc-server kini punya fungsi operasional nyata



---

## Backup Server, Proxmox Backup Server (pbs-server, 192.168.0.25)

### Persiapan VM
```bash
# Provision VM baru pakai ISO Debian 13 (trixie) netinst BUKAN Ubuntu
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
- **General type of mail configuration**: pilih `Internet with smarthost` best practice-nya notifikasi backup/alerting dikirim independen dari infrastruktur yang dipantau.
- **System mail name**: `homelab.internal`
- **SMTP relay host**: `[smtp.gmail.com]:587` tanda kurung siku wajib biasanya maksa connect langsung ke host, bukan cari MX record, port 587 untuk STARTTLS.

### Setup notifikasi Gmail (SASL auth)
```bash
apt install libsasl2-modules -y
```
Modul tambahan yang dibutuhkan Postfix untuk autentikasi SASL ke SMTP luar (Gmail).

```bash
nano /etc/postfix/sasl_passwd
```
Isi 1 baris: `[smtp.gmail.com]:587  emailkamu@gmail.com:appPassword16digit`. App Password didapat dari [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords) wajib aktifkan 2-Step Verification dulu di akun Google, Gmail tidak menerima password akun biasa untuk SMTP pihak ketiga.

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
Restart Postfix lalu kirim email test. **Catatan:** kalau muncul warning `overriding earlier entry: smtp_tls_security_level` itu cuma info, bukan error fatal, karena Debian sudah punya baris default `smtp_tls_security_level=may` bawaan yang bentrok sama baris baru kita (`encrypt`). Postfix otomatis pakai yang paling akhir ditulis, email tetap terkirim normal. Bisa dibersihin nanti dengan `grep -n smtp_tls_security_level /etc/postfix/main.cf` 

### Konfigurasi PBS (web UI)
```
https://192.168.0.25:8007
```
Login pakai `root` + password Debian. Warning certificate self-signed itu normal, klik "Advanced > Proceed".

- **Datastore > Add Datastore** nama `homelab-backup`, path penyimpanan misal `/mnt/backup`.
- Login ke Proxmox VE (`https://192.168.0.10:8006`) → **Datacenter > Storage > Add > Proxmox Backup Server** isi Server `192.168.0.25`, Datastore `homelab-backup`, username `root@pam`, password root PBS.
- **Datacenter > Backup > Add** pilih storage PBS, centang semua VM, atur jadwal semisal tiap jam 2 pagi, mode **Snapshot** VM tetap jalan saat backup.

### Test backup & restore
```
Klik VM (misal dns-server) > Backup > Backup now, pilih storage PBS.
```
Test manual tanpa nunggu jadwal. Cek hasilnya muncul di datastore PBS.

```
Restore backup ke VM ID baru (misal 199, bukan ID asli).
```
**Step paling penting** backup yang belum pernah dites restore-nya nggak bisa dipercaya. Kalau VM hasil restore boot normal dan semua service (termasuk resolusi DNS ke domain lain) masih jalan, berarti backup beneran valid. Hapus VM test setelah verifikasi selesai.

---

## VPN WireGuard (Gagal) Pindah ke Tailscale (vpn-server, 192.168.0.26)

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
Aktifkan IP forwarding, wajib supaya VPN server bisa nerusin paket ke LAN 192.168.0.x.

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg show
```
Aktifkan service. `wg show` menampilkan status interface + baris "latest handshake" (indikator utama koneksi client beneran jalan atau tidak).

**Setup pendukung (di luar VM):**
- Router D-Link DIR-612: menu **Virtual Server** (bukan "Port Forwarding") isi Private IP `192.168.0.26`, Protocol **UDP**, port `51820`.
- Cek CGNAT dulu sebelum setup port forwarding: bandingkan WAN IP di halaman status router vs hasil search "what is my ip" kalo beda, port forwarding tidak akan pernah berfungsi.
- DDNS via No-IP: buat hostname baru di [my.noip.com](https://my.noip.com), **wajib centang "Enable Dynamic DNS"**, lalu isi hostname/username/password itu di menu DDNS router. Update period diisi `5` (menit).

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
sudo wg show                                    
# tidak ada "latest handshake" sama sekali
sudo tcpdump -i ens18 udp port 51820 -n         
# 0 packets captured saat client coba connect
```
Dicoba ganti port ke `443` tetap 0 packets. Test port TCP lain (8080) via canyouseeme.org **berhasil**, artinya port forwarding & jaringan rumah normal untuk TCP. Firewall Proxmox (VM/Node/Datacenter) dan UFW di VM semua sudah dicek nonaktif. **Kesimpulan:** ISP/operator seluler kemungkinan besar memblokir trafik UDP pada port non-standar dan bukan salah konfigurasi, tapi pembatasan jaringan di luar kendali. WireGuard manual tidak bisa dipakai dalam kondisi ini.

### Percobaan 2: Tailscale (berhasil)

```bash
sudo systemctl stop wg-quick@wg0
sudo systemctl disable wg-quick@wg0
```
Matikan WireGuard yang gagal sebelum pindah solusi.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```
Install Tailscale, software ini dibangun di atas protokol WireGuard yang sama, tapi punya NAT traversal otomatis (server relay DERP) sehingga tidak butuh port forwarding sama sekali.

```bash
sudo tailscale up --advertise-routes=192.168.0.0/24
```
Login via link yang muncul lewat autentikasi akun Google, lalu **advertise** subnet LAN ini yang bikin device Tailscale lain bisa "menembus" ke jaringan 192.168.0.x lewat vpn-server sebagai gateway.

```
login.tailscale.com/admin/machines
```
Approve subnet route yang di-advertise tadi (device vpn-server > Edit subnet routes > centang `192.168.0.0/24` > Save)

**Install Tailscale di HP/laptop**, login pakai akun yang sama, nantinya subnet routes otomatis keterima di OS tersebut.

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
- `vpn-server` dan `dns-server` diset **Start at boot = Yes** di Proxmox, supaya remote access selalu bisa dilakukan kapan saja tanpa perlu nyalain manual.
- VM lain (web, mail, DC, file, PBS) dinyalakan manual sesuai kebutuhan lewat Proxmox web UI setelah terhubung VPN, menghemat resource RAM saat tidak dipakai.

---

## Monitoring lewat Zabbix (monitor-server, 192.168.0.27)

### Provisioning & Zabbix Server
```bash
sudo hostnamectl set-hostname monitor-server
sudo nano /etc/netplan/50-cloud-init.yaml   # IP statis IsiIP, nameserver ke DomainUtama(DNS-SERVER)
sudo netplan apply
sudo apt update && sudo apt upgrade -y
```
```bash
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo apt update
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent mariadb-server -y
```

### Setup database
```bash
sudo mysql_secure_installation
sudo mysql -u root -p
```
```sql
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'passwordkuat';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```
`utf8mb4` wajib untuk dukungan karakter penuh. `log_bin_trust_function_creators` diaktifkan sementara agar proses import schema tidak kena error terkait binary logging.
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -u zabbix -p zabbix
```
Import schema awal(agak lamaa).
```bash
sudo mysql -u root -p
```
```sql
set global log_bin_trust_function_creators = 0;
quit;
```
Matikan lagi setelah import selesai, biar tidak permanen aktif.

### Konfigurasi zabbix_server.conf
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
Isi (hapus tanda `#` di depan tiap baris):
```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=passwordkuat
```
```bash
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
sudo systemctl status zabbix-server
```
Kalau `failed`, cek log: `sudo tail -30 /var/log/zabbix/zabbix_server.log` penyebab tersering salah password di config.

### Setup wizard web UI
```
http://192.168.0.27/zabbix
```
Ikuti wizard: pre-requisites check (PHP timezone dll), Configure DB connection (host `localhost`, database `zabbix`, user `zabbix`, password sesuai config), Zabbix server details (biarkan default). Login pertama pakai `Admin`/`zabbix` **wajib langsung ganti password default** setelah berhasil masuk (Profile > ganti password).

### Install Zabbix Agent di tiap VM yang dipantau
Untuk VM Ubuntu (dns, web, mail, dc, file, vpn):
```bash
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_6.4-1+ubuntu24.04_all.deb
sudo apt update
sudo apt install zabbix-agent -y
```
Untuk pbs-server (Debian 13 Zabbix 6.4 belum ada build trixie, pakai repo 7.0):
```bash
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.0+debian13_all.deb
sudo dpkg -i zabbix-release_latest_7.0+debian13_all.deb
sudo apt update
sudo apt install zabbix-agent -y
```
Konfigurasi sama di semua VM:
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```
```
Server=192.168.0.27
ServerActive=192.168.0.27
Hostname=<nama-vm>
```
```bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
sudo systemctl status zabbix-agent
```

### Registrasi host & alerting (web UI)
- **Data collection > Hosts > Create host** Host name harus PERSIS sama dengan `Hostname=` di agent, template **Linux by Zabbix agent**, interface Agent diisi IP VM masing-masing, port `10050`.
- **Alerts > Media types > Email** SMTP `smtp.gmail.com:587`, STARTTLS, autentikasi Gmail App Password.
- **Users > Admin > Media** tambahkan email tujuan notifikasi.
- **Alerts > Actions > Trigger actions** pastikan action default aktif.

### Verifikasi
Ketujuh VM lain terdaftar dan mulai melaporkan data (CPU, RAM, disk, status service) di dashboard Zabbix. Alerting email diuji dan berhasil mengirim notifikasi.

---

*(Command reference lengkap: DNS, Web, Mail, Directory Server, File Server, Backup Server, VPN, Monitoring)*