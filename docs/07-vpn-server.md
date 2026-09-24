# VPN Remote Access

**VM:** vpn-server `192.168.0.26`, menggunakan Ubuntu Server 24.04 (Tailscale)

## Tujuan

Memungkinkan akses ke seluruh jaringan internal dari luar rumah dan mensimulasikan kebutuhan *remote worker* yang harus tetap bisa mengakses file server, webmail, dan layanan internal lain tanpa berada secara fisik di jaringan kantor.

## Keputusan Desain Kunci

Bagian ini paling bernilai secara pembelajaran karena melibatkan *pivot* solusi berdasarkan bukti dari lapangan, bukan sekadar mengikuti rencana awal.

**Rencana awal: WireGuard manual**, lengkap dengan port forwarding di router rumah (D-Link DIR-612) dan Dynamic DNS (No-IP) untuk mengatasi IP publik yang berubah-ubah.

**Hasilnya gagal** setelah diagnosis sistematis, disimpulkan ISP/operator seluler memblokir trafik UDP pada port non-standar, bukan kesalahan konfigurasi.

**Solusi akhir: Tailscale** sebagai subnet router pada VM yang sama, memakai protokol WireGuard yang sama di bawahnya, namun dengan NAT traversal otomatis (server relay) yang tidak membutuhkan port forwarding sama sekali.

## Proses Diagnosis WireGuard

| Langkah Cek | Hasil |
|---|---|
| `wg show` cek *latest handshake* | Tidak pernah ada handshake |
| `tcpdump udp port 51820` saat client connect | 0 packets captured |
| Ganti port ke 443, ulangi tcpdump | Tetap 0 packets |
| Test port TCP lain via canyouseeme.org | **Berhasil** membuktikan port forwarding & jaringan rumah normal untuk TCP |
| Bandingkan WAN IP router vs IP publik dari luar | Sama persis bukan kasus CGNAT |

Kombinasi hasil ini mengarah ke satu kesimpulan: masalahnya spesifik pada UDP, di luar kendali konfigurasi lokal atau keterbatasan pengetahuan.

## Konfigurasi Inti atau solusi akhir yang dilakukan

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=192.168.0.0/24
```

Subnet route di-*approve* manual di [Tailscale Admin Console](https://login.tailscale.com/admin/machines). Split DNS dikonfigurasi agar domain internal tetap resolve dari perangkat remote (nameserver `192.168.0.20`, dibatasi ke domain `homelab.internal`).

## Verifikasi

```bash
sudo tailscale status
sudo tailscale ping <device>
```
Diuji dari HP dalam mode data seluler dan device lain dengan jaringan yang berbeda, berhasil mengakses `https://demo.homelab.internal`, `familypet.com` dan melakukan SSH ke semua VM yang menyala dari luar jaringan rumah sepenuhnya.

## Screenshot

Akses Dari HP ke website demo.homelab.internal

![VPN dari data seluler](/images/vpn-server1.jpeg)

SSH ke VM monitor-server

![VPN dari data seluler](/images/vpn-server2.png)

*HP maupun laptop dengan jaringan yang berbeda berhasil terhubung lewat Tailscale dan mengakses layanan internal, bukti remote access berfungsi dari luar jaringan rumah.*
