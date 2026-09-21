# Web Server

**VM:** web-server `192.168.0.21`, menggunakan Ubuntu Server 24.04 (Nginx + PHP-FPM)

## Tujuan

Meng-host portal internal kantor dan situs demo, sekaligus jadi tempat latihan konsep *virtual hosting* yang dimana satu server Nginx melayani banyak domain sekaligus.

## Keputusan Desain Kunci

- **Multi-site via server block**, bukan multi-IP, satu Nginx melayani beberapa domain (`demo.homelab.internal`, `familypet.com`) sekaligus, dibedakan lewat `server_name`. IP terpisah per domain hanya diperlukan untuk kasus khusus (load balancing, isolasi ketat).
- **Dummy `default_server`** yang me-*return* `444` untuk menolak akses langsung via IP, fungsinya untuk mencegah *information leak* ke siapapun yang scan IP tanpa tahu nama domainnya.
- **HTTPS self-signed** untuk semua domain, Private CA sengaja tidak dipasang.

## Konfigurasi Inti

```nginx
# Dummy default_server — security hardening
server {
    listen 80 default_server;
    server_name _;
    return 444;
}

# Server block per-domain, contoh familypet.com
server {
    listen 443 ssl;
    server_name familypet.com;
    root /var/www/html;
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

## Tantangan & Solusi

| Tantangan | Solusi |
|---|---|
| `duplicate default server` error saat menambah domain baru | Hanya boleh 1 file config yang punya flag `default_server` di seluruh `sites-enabled/` |
| Closing brace `}` ter-*comment* tanpa sengaja atau adanya typo yang dimana bikin syntax error | Diperiksa dengan `nginx -t` dan `cat -n` untuk melacak baris persis |

## Verifikasi

```bash
# cek syntax config valid
sudo nginx -t                   
# isolasi antar-domain atau bisa langsung ke browser tulisan nama domain               
curl -I -H "Host: familypet.com" http://192.168.0.21   
# harus ditolak (444) via IP langsung atau bisa langsung ke browser
curl -I http://192.168.0.21                    
```

## Screenshot
*192.168.0.21*
![HTTPS multi-domain](/images/web-server3.png)

*demo.homelab.internal*
![HTTPS multi-domain](/images/web-server2.png)

*familypet.com*
![HTTPS multi-domain](/images/web-server1.png)
*Browser mengakses 2-3 domain berbeda (`demo.homelab.internal`, `familypet.com`) masing-masing dengan HTTPS aktif (gembok self-signed).*