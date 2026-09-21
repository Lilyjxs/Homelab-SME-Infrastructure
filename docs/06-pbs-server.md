# Backup Server

**VM:** pbs-server `192.168.0.25`, menggunakan Debian 13 (Proxmox Backup Server)

## Tujuan

Menyediakan backup terjadwal dan *deduplicated* untuk seluruh VM, plus jalur notifikasi yang independen dari infrastruktur yang dipantau untuk memastikan kegagalan tetap diketahui meski komponen lain termasuk mail server sendiri sedang down.

## Keputusan Desain Kunci

- **Notifikasi lewat Gmail SMTP, bukan mail-server internal** kesalahan arsitektur yang umum terjadi adalah *alerting system* yang bergantung pada infrastruktur yang sama dengan yang diawasi; jika mail-server down, notifikasi kegagalan justru tidak akan pernah sampai.
- **Retention policy (prune) wajib diatur eksplisit** PBS tidak otomatis menghapus backup lama, lalu tanpa *prune* storage bisa membengkak tanpa batas.

## Konfigurasi Inti

```bash
echo "deb http://download.proxmox.com/debian/pbs trixie pbs-no-subscription" > /etc/apt/sources.list.d/pbs.list
apt install proxmox-backup-server -y
```

Masuk ke file sasl_passwd atur notifikasi via Gmail
```bash
nano /etc/postfix/sasl_passwd
```
Isi config dengan email serta password yang udah didapatkan
```
[smtp.gmail.com]:587    email@gmail.com:app-password
```

Retention policy: **Keep Last: 1** hanya backup terakhir yang disimpan per VM, backup lama otomatis dihapus setelah backup baru berhasil.

## Tantangan & Solusi

| Tantangan | Solusi |
|---|---|
| Backup berpotensi menumpuk tanpa batas jika tidak diatur sesuai kondisi yang dimau | Prune policy dikonfigurasi eksplisit, disadari juga bahwa *prune* saja tidak langsung membebaskan disk, perlu *Garbage Collection* untuk benar-benar menghapus chunk data yang sudah tak terpakai |
| Notifikasi Gmail perlu autentikasi, beda dari relay internal yang bebas kredensial | Setup App Password Gmail + SASL auth di Postfix |

## Verifikasi

Backup manual dijalankan, lalu **diuji restore penuh** ke VM ID baru (bukan menimpa yang asli), termasuk verifikasi bahwa layanan di dalam VM hasil restore tetap berfungsi normal, sebelum VM uji dihapus. Backup yang belum pernah diuji restore-nya dianggap tidak bisa dipercaya.

## Screenshot

Riwayat Backup
![Dashboard PBS](/images/pbs-server1.png)

Backup Content
![Dashboard PBS](/images/pbs-server2.png)
*Dashboard Proxmox Backup Server menampilkan riwayat backup tiap VM, dan hasil uji restore ke VM ID baru yang berhasil boot normal.*