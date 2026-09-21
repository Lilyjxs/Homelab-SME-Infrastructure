# Directory Server

**VM:** dc-server `192.168.0.23`, menggunakan Ubuntu Server 24.04 (Samba Active Directory Domain Controller)

## Tujuan

Menyediakan identity management terpusat (realm `CORP.HOMELAB.INTERNAL`) satu database user & grup yang bisa dipakai berbagai layanan lain untuk autentikasi, menggantikan system "user Linux lokal per-server" yang sulit di-*maintain* seiring bertambahnya jumlah karyawan/service.

## Keputusan Desain Kunci

- **Domain AD dipisah dari domain infrastruktur** (`corp.homelab.internal`, bukan `homelab.internal` langsung) mengikuti best practice standar. Samba AD DC wajib menjadi otoritas DNS penuh untuk domainnya sendiri, sehingga penyatuan domain akan membentur BIND9 yang sudah ada.
- **Time sync (chrony) sebagai prasyarat wajib** Kerberos, komponen inti AD, gagal total jika ada selisih waktu antar-server lebih dari beberapa menit.

## Konfigurasi Inti

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
# Realm: CORP.HOMELAB.INTERNAL | Domain: CORP | DNS backend: SAMBA_INTERNAL
```

Delegasi dari BIND9 (`db.domain`):
```
corp                              IN  NS  dc-server.corp.homelab.internal.
dc-server.corp.homelab.internal.  IN  A   192.168.0.23
```

## Tantangan & Solusi

| Tantangan | Solusi |
|---|---|
| Provisioning gagal: `'realm =' was not specified` | `smb.conf` bawaan paket standalone bentrok dengan proses provisioning, solusi dihapus, dijalankan ulang |
| Glue record tidak terbaca meski data terlihat benar | FQDN tanpa titik penutup dianggap nama relatif oleh BIND, wajib diakhiri titik (`.`) |
| Query domain AD dari BIND9 selalu NXDOMAIN meski delegasi sudah benar | Forwarder global BIND9 (`8.8.8.8`) membajak query sebelum sempat memakai delegasi lokal, solusi diperbaiki dengan blok `type forward` khusus |

## Catatan

Directory server ini awalnya berdiri sendiri atau standalone sebagai bukti konsep user & grup AD dibuat, tapi tidak ada layanan lain yang memverifikasi login lewatnya, sehingga belum memberi manfaat operasional nyata. Ini diperbaiki lewat integrasi domain join ke File Server yang nanti bisa dilihat pada [05-file-server.md](/docs//05-file-server.md).

## Verifikasi

```bash
# test autentikasi Kerberos
kinit administrator@CORP.HOMELAB.INTERNAL 
 # cek user terdaftar  
samba-tool user list                    
# cek SRV record AD    
host -t SRV _ldap._tcp.corp.homelab.internal 
```

## Screenshot

![Kerberos & user AD](/images/dc-server.png)
*Terminal menampilkan `kinit` berhasil mendapat tiket Kerberos, dan `samba-tool user list` menunjukkan user/grup yang terdaftar di domain `CORP.HOMELAB.INTERNAL`.*