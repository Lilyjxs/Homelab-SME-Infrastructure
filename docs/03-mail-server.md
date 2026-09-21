# Mail Server

**VM:** mail-server `192.168.0.22`, menggunakan Ubuntu Server 24.04 (Postfix + Dovecot + Roundcube)

## Tujuan

Menyediakan layanan email internal lengkap seperti kirim/terima pesan antar-karyawan lewat SMTP/IMAP, plus antarmuka webmail sebagai simulasi kebutuhan komunikasi kantor tanpa bergantung pada penyedia email eksternal.

## Keputusan Desain Kunci

- **Internal-only**, bukan relay ke internet, `mynetworks` dibatasi ke subnet lokal saja, bukan `0.0.0.0/0`. Menghindari server jadi *open relay* untuk spam, meski modul referensi menyarankan pengaturan yang lebih longgar.
- **Domain email terpisah dari hostname server** alamat email pakai `@homelab.internal`, sedangkan `mail.homelab.internal` murni hostname teknis. Kesalahan umum: mengira keduanya bisa dipakai bergantian.

## Konfigurasi Inti

```
# /etc/postfix/main.cf
home_mailbox = Maildir/
mynetworks = 127.0.0.0/8 192.168.0.0/24
```

```
# /etc/dovecot/conf.d/10-mail.conf
mail_location = maildir:~/Maildir
```

Masuk ke file config.inc.php
```bash
nano /etc/roundcube/config.inc.php
```
```php
$config['imap_host'] = ["ssl://mail.homelab.internal:993"];
$config['mail_domain'] = 'homelab.internal';
```

## Tantangan & Solusi

| Tantangan | Solusi |
|---|---|
| Email terkirim tapi tidak diterima jika alamat pakai `@mail.homelab.internal` | Domain itu tidak terdaftar di `mydestination` Postfix hanya `@homelab.internal` yang valid |
| Roundcube gagal konek ke Dovecot: `getaddrinfo failed` | Mail-server sendiri belum diarahkan memakai DNS internal (`192.168.0.20`) sebagai resolver |
| Identity pengirim otomatis `user@mail.homelab.internal`, bukan `@homelab.internal` | Ditambahkan `$config['mail_domain']` di Roundcube agar identity baru otomatis benar |
| `SSL_accept() failed: unknown ca` setelah SSL Dovecot diaktifkan | PHP menolak verifikasi certificate self-signed, hasrus nambahin `imap_conn_options` dengan `verify_peer => false` |

## Verifikasi

```bash
# Cara cek log
tail -f /var/log/mail.log                             
```
Login Roundcube via `https://mail.homelab.internal`, kirim & terima antar-user internal berhasil.

## Screenshot

Form Login 
![Roundcube inbox](/images/mail1.png)

Sent Mail
![Roundcube inbox](/images/mail2.png)

Rrceived Mail
![Roundcube inbox](/images/mail3.png)
*Antarmuka webmail Roundcube menampilkan email yang berhasil dikirim dan diterima antar-user internal (`@homelab.internal`).*