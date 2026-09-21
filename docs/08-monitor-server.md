# Monitoring

**VM:** monitor-server `192.168.0.27`, menggunakan Ubuntu Server 24.04 (Zabbix)

## Tujuan

Menyediakan visibilitas terpusat atas kondisi seluruh infrastruktur (CPU, RAM, disk, status service) dan alerting otomatis, serta komponen penutup yang mengubah infrastruktur dari "sekumpulan server yang jalan" menjadi "infrastruktur yang bisa diawasi", sesuai konsep monitoring/NOC di kantor sungguhan.

## Keputusan Desain Kunci

- **Arsitektur agent-server** satu Zabbix Server memantau seluruh 7 VM lain lewat Zabbix Agent yang terinstall masing-masing, melapor secara berkala.

## Konfigurasi Inti

```bash
# Zabbix Server
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf mariadb-server -y
```

Isi di setiap file pada VM yang mau dipantau
```bash
nano /etc/zabbix/zabbix_agentd.conf
```
config yang harus diisi
```
Server=192.168.0.27
ServerActive=192.168.0.27
Hostname=<nama-vm>
```

## Verifikasi

Ketujuh VM lain (dns, web, mail, dc, file, pbs, vpn) terdaftar di **Data collection > Hosts** dengan template *Linux by Zabbix agent*, dan mulai melaporkan data. Alerting email (SMTP Gmail, pola yang sama seperti notifikasi PBS) dikonfigurasi dan diuji berhasil mengirim notifikasi.

## Screenshot

Tampilan Monitoring Host
![Dashboard Zabbix](/images/zabbix1.png)

Tampilan Dashboard
![Dashboard Zabbix](/images/zabbix2.png)

Tampilan Notifikasi Email
![Dashboard Zabbix](/images/zabbix3.png)

*Dashboard Zabbix menampilkan seluruh 8 VM dalam status termonitor, lengkap dengan metrik dasar (CPU, RAM, disk) tiap host.*