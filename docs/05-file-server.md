# File Server

**VM:** file-server `192.168.0.24`, menggunakan Ubuntu Server 24.04 - Samba (Active Directory domain member)

## Tujuan

Menyediakan network share per-divisi dengan kontrol akses berbasis grup yang mensimulasikan kebutuhan kolaborasi file kantor sekaligus jadi tempat AD (dc-server) benar-benar dipakai secara operasional.

## Keputusan Desain Kunci

- **Dua fase implementasi disengaja**: dimulai dari standalone menggunakan grup Linux lokal untuk memahami dasar Samba file-sharing, baru kemudian di-*upgrade* jadi domain member AD lewat Winbind dengan pendekatan bertahap ini membantu memisahkan kompleksitas "cara kerja file sharing" dari "cara kerja integrasi AD".
- **Permission berbasis grup AD** (`CORP\IT`, `CORP\Finance`, `CORP\Domain Users`), bukan lagi grup Linux lokal, begitu domain join selesai, kepemilikan folder (`chgrp`) diarahkan ke GID grup AD.

## Konfigurasi Inti

tambahkan pada file smb.conf, cari bagian [global], setelah domain join
```bash
nano /etc/samba/smb.conf 
```

selanjutnya isi config seperti ini:
```
workgroup = CORP
realm = CORP.HOMELAB.INTERNAL
security = ads
idmap config * : backend = tdb
idmap config CORP : backend = rid
idmap config CORP : range = 10000-999999
```

Ganti setiap divisi seperti ini di setiap valid users: `@"CORP\DivisiYangMauDiisi`
```
[IT]
   path = /srv/samba/it
   valid users = @"CORP\IT"
```

```bash
sudo net ads join -U administrator   # domain join
```

## Tantangan & Solusi

| Tantangan | Solusi |
|---|---|
| User AD tidak otomatis bisa dipakai login ke share meski sudah ada di dc-server | Butuh domain join eksplisit (winbind + PAM/NSS) karena AD dan file-server awalnya dua sistem user yang sepenuhnya terpisah |
| `chgrp` dengan nama grup AD kadang gagal | Fallback memakai GID numerik hasil `getent group "CORP\<grup>"` |

## Verifikasi

Cara cek user AD terlihat dari file-server
```bash
wbinfo -u                                    
```
Test dari sisi Linux
```bash                     
smbclient //localhost/IT -U dara            
```
```cmd
net use Z: \\192.168.0.24\IT /user:CORP\dara
```
Diuji dengan beberapa user AD (keanggotaan grup berbeda) yang dimana masing-masing hanya bisa mengakses share sesuai grupnya, dikonfirmasi baik akses sukses maupun akses ditolak berjalan sesuai desain.

## Screenshot

User dara group IT akses ke folder Finance
![Akses share via Windows](/images/file-server1.png)

User dara group IT akses ke folder IT
![Akses share via Windows](/images/file-server2.png)

Hasil
![Akses share via Windows](/images/file-server3.png)


*Windows Explorer menampilkan akses berhasil ke folder sesuai grup AD user, dan percobaan akses ke folder lain (di luar grupnya) ditolak dan membuktikan isolasi permission berfungsi.*