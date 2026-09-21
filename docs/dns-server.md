# DNS Server

**VM:** dns-server `192.168.0.20`, Ubuntu Server 24.04 (BIND9)

## Tujuan

Menjadi otoritas DNS internal untuk seluruh infrastruktur, resolusi nama domain (`homelab.internal`), reverse lookup, dan MX record untuk mail server, tanpa bergantung pada DNS publik untuk komunikasi antar-service internal.

## Keputusan Desain Kunci

- **Domain internal (`homelab.internal`)** dipilih setelah dua percobaan gagal: `.local` memicu konflik dengan mDNS, dan `.id` bentrok dengan domain publik nyata.
- **Domain kedua (`familypet.com`)** dibuat sebagai *zone* terpisah (bukan subdomain) untuk simulasi hosting multi-domain, murni internal, meniru nama domain publik untuk latihan.
- **1 PTR per IP** jadi reverse zone dirapikan agar setiap IP hanya punya satu nama kanonik, sesuai konvensi standar (awalnya sempat ada 3 PTR untuk 1 IP).

## Konfigurasi Inti

```
; db.domain (forward zone)
@       IN      NS      homelab.internal.
@       IN      A       192.168.0.20
@       IN      MX      10      mail.homelab.internal.
mail    IN      A       192.168.0.22
web     IN      A       192.168.0.21
```

```
; db.homelab (reverse zone)
20      IN      PTR     ns.homelab.internal.
22      IN      PTR     mail.homelab.internal.
```

Domain Active Directory (`corp.homelab.internal`) di-*delegasikan* ke `dc-server` lewat NS + glue record, bisa dilihat [directory-server.md](./directory-server.md) untuk detail penuh.

## Tantangan & Solusi

| Tantangan | Solusi |
| Windows tidak resolve domain internal meski DNS sudah diset manual | Windows ternyata memakai resolver IPv6 link-local dari router; solusi: matikan IPv6 di adapter atau pastikan urutan prioritas DNS benar |
| Query domain yang sudah didelegasikan tetap NXDOMAIN meski data zone sudah benar | Forwarder global (`8.8.8.8`) membajak query sebelum delegasi lokal sempat dipakai — diperbaiki dengan blok `type forward` khusus per-domain |

## Verifikasi

```bash
dig corp.homelab.internal
nslookup mail.homelab.internal
nslookup -type=MX homelab.internal
```

## Screenshot

![Hasil resolusi DNS](./images/dns.png)
*Terminal menampilkan `nslookup`/`dig` berhasil resolve `homelab.internal`, `web.homelab.internal`, `mail.homelab.internal`, dan `corp.homelab.internal`, `familypet.com` ke IP yang benar.*