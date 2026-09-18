# ONU Speedtest — GenieACS

Dashboard uji kecepatan ONU lewat TR-143, terintegrasi langsung dengan GenieACS NBI. Satu file HTML, tanpa dependensi dan tanpa build step.

Pilih ONU, klik sekali, dapat angka unduh dan unggah dalam satu rangkaian — tanpa datang ke lokasi.

![screenshot](docs/screenshot.png)

## Fitur

- Uji unduh dan unggah berurutan otomatis, atau salah satu saja
- Gauge analog dengan animasi jarum
- Daftar perangkat dengan pencarian dan status inform terakhir
- Kode error TR-143 diterjemahkan ke penyebab konkret
- Login tervalidasi server (basic auth nginx), tanpa kredensial di dalam berkas
- Log proses per langkah untuk penelusuran masalah

## Yang dibutuhkan

- GenieACS 1.2+
- nginx (`nginx-full` kalau mau uji unggah — butuh modul DAV)
- ONU yang mendukung TR-143 (`DownloadDiagnostics` / `UploadDiagnostics`)

Cek dukungan ONU dulu: buka device di GenieACS → **All parameters** → filter `Diagnostics`. Kalau node-nya tidak ada, ONU tersebut tidak bisa diuji.

## Pemasangan singkat

**1. File uji dan proksi NBI**

```bash
apt install -y nginx apache2-utils
mkdir -p /var/www/html/test /var/www/html/speedtest
truncate -s 200M /var/www/html/test/200MB.bin
htpasswd -c /etc/nginx/.htpasswd namauser
```

Di dalam blok `server` pada `/etc/nginx/sites-available/default`:

```nginx
location /nbi/ {
    auth_basic "Speedtest";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:7557/;
    proxy_read_timeout 900s;
}
```

```bash
nginx -t && systemctl reload nginx
```

**2. Provision dan preset di GenieACS**

Ada di [`provisions/`](provisions/). Sesuaikan alamat server di dalamnya, lalu buat lewat **Admin → Provisions**.

| Preset | Precondition | Events | Provision |
|---|---|---|---|
| `speedtest_start` | `Tags.speedtest = true` | *(kosong)* | `speedtest_start` |
| `speedtest_up_start` | `Tags.speedtestup = true` | *(kosong)* | `speedtest_up_start` |
| `speedtest_read` | `Tags.speedtestwait = true` | `8 DIAGNOSTICS COMPLETE` | `speedtest_read` |

**3. Halaman**

Salin `index.html` ke `/var/www/html/speedtest/`, buka `http://<IP-SERVER>/speedtest/`.

Logo opsional: taruh `logo.png` atau `logo.svg` di folder yang sama.

## Uji manual dulu

Sebelum pakai dashboard, pastikan pondasinya jalan. Tag satu ONU dengan `speedtest` lalu **Summon**, tunggu 30 detik, cek `DiagnosticsState`:

| Nilai | Artinya |
|---|---|
| `Complete` / `Completed` | Berhasil |
| `Requested` (nyangkut) | ONU tidak bisa menarik file — cek nginx dan firewall |
| `Error_InitConnectionFailed` | Parameter `Interface` salah, atau port 80 tertutup |
| `Error_NoResponse` | nginx mati atau URL salah |

Dashboard hanya memicu dan memanen hasil. Kalau uji manual gagal, dashboard juga pasti gagal.

## Untuk unggah

Butuh nginx dengan modul DAV:

```bash
apt install -y nginx-full
mkdir -p /var/www/upload /var/tmp/nginx_up
chown -R www-data:www-data /var/www/upload /var/tmp/nginx_up
echo '*/10 * * * * root find /var/www/upload /var/tmp/nginx_up -type f -mmin +20 -delete' \
  > /etc/cron.d/bersihkan-upload
```

```nginx
location /upload/ {
    alias /var/www/upload/;
    dav_methods PUT DELETE;
    client_max_body_size 0;
    client_body_temp_path /var/tmp/nginx_up;
    create_full_put_path on;
    dav_access user:rw group:rw all:r;
}
```

Verifikasi: `curl -X PUT --data-binary @/etc/hostname http://<IP-SERVER>/upload/coba.txt -i` harus `201 Created`.

## Catatan

**Ukuran file uji.** Durasi transfer harus lewat dari TCP slow start. Di bawah 5 detik angkanya tidak bisa dipercaya. 200 MB cukup untuk kebanyakan kasus; 1 GB baru masuk akal di atas 300 Mbps.

**Hasilnya throughput CPU ONU, bukan kapasitas jalur.** ONU kelas rumahan sering mentok di 200–300 Mbps walaupun paketnya 1 Gbps.

**Unggah selalu lebih lambat.** Upstream GPON itu TDMA, ONU menunggu jatah slot dari OLT. Rasio 4:1 atau 5:1 normal.

**Jangan jalankan serentak** ke banyak ONU di PON yang sama — ini trafik sungguhan di jaringan produksi.

**Amankan NBI.** Lewat `/nbi/` siapa pun bisa reboot sampai factory reset seluruh ONU. Jangan pernah dibuka ke internet.

## Tulisan lengkap

Latar belakang, penjelasan TR-143, dan jebakan yang ditemui selama membangunnya: [farabingonfig.wordpress.com](https://farabingonfig.wordpress.com)

## Lisensi

MIT
