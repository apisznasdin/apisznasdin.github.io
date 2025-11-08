---
layout: post
title: "rclone & Google Drive: Panduan Backup Direktori /home"
date: 2025-11-06 23:15:00 +0800
categories: [linux, tutorial, archlinux, rclone, systemd]
tags: [google-drive, rclone, systemd, mount, linux, tutorial]
---

Salah satu cabaran bagi pengguna Linux ialah ketiadaan klien Google Drive rasmi. Walaupun ada klien pihak ketiga seperti Insync (berbayar), terdapat satu cara yang lebih berkuasa, percuma, dan "native" untuk pengguna Linux, iaitu menggunakan **rclone**.

Daripada menyalin (sync) semua fail, kaedah ini akan "mount" (melekapkan) folder Google Drive anda terus ke dalam direktori `/home` anda. Contohnya, folder `Documents` di Google Drive akan muncul di `~/Documents` seolah-olah ia adalah folder tempatan.

Dalam tutorial ini, kita akan menggunakan `rclone` dan `systemd` (sebagai servis pengguna) untuk memasang folder `Documents`, `Pictures`, dan `Videos` dari Google Drive secara automatik semasa log masuk.

![Remote Google Drive mount point]({{ "assets/images/dolphin_remote.png" | relative_url }})

---

## 1\. Persediaan Awal

Sebelum kita boleh membuat servis `systemd`, ada beberapa perkara yang perlu disediakan.

### Pasang Pakej yang Diperlukan
Kita memerlukan `rclone` untuk berhubung dengan Google Drive dan `fuse3` untuk membolehkan proses *mount* di ruang pengguna (userspace).

```bash
# Untuk pengguna Arch Linux
sudo pacman -S rclone fuse3
````

### Konfigurasi rclone

Anda perlu mengkonfigurasi `rclone` dan menyambung ke akaun [Google Drive](https://rclone.org/drive/) anda.

```bash
rclone config
```

Ikut arahan pada skrin:

1.  Pilih `n` (New remote).
2.  Beri nama, contohnya: `gdrive`.
3.  Pilih `drive` (Google Drive) dari senarai.
4.  Ikut langkah-langkah untuk memberi kebenaran (ia akan membuka pelayar web).
5.  Simpan konfigurasi.

### Cipta Direktori Mount

Pastikan folder tempatan dalam direktori `/home` anda wujud.

```bash
mkdir -p ~/Documents
mkdir -p ~/Pictures
mkdir -p ~/Videos
```

-----

## 2\. Membuat Servis systemd (User)

Kita akan menggunakan servis **user** `systemd`, bukan servis *system*. Ini penting kerana servis *user* akan berjalan dengan kebenaran (permission) anda dan hanya aktif apabila anda log masuk.

Fail servis ini mesti diletakkan di `~/.config/systemd/user/`.

### Servis 1: gdrive-documents.service

Cipta fail pertama untuk folder `Documents`.

**Fail:** `~/.config/systemd/user/gdrive-documents.service`

```bash
nano ~/.config/systemd/user/gdrive-documents.service
```

**Kandungan:**

```ini
[Unit]
Description=Google Drive (Documents) Mount
# Pastikan servis ini bermula selepas rangkaian sedia
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
# %h ialah pembolehubah systemd untuk direktori /home anda
ExecStart=/usr/bin/rclone mount gdrive:Documents %h/Documents \
  --vfs-cache-mode writes \
  --poll-interval 1m
ExecStop=/usr/bin/fusermount3 -u %h/Documents
Restart=on-failure
RestartSec=5

[Install]
# Mula servis ini secara automatik semasa log masuk
WantedBy=default.target
```

### Servis 2: gdrive-pictures.service

Sekarang, kita buat fail kedua untuk `Pictures`. Prosesnya sama, hanya tukar nama folder.

**Fail:** `~/.config/systemd/user/gdrive-pictures.service`

```bash
nano ~/.config/systemd/user/gdrive-pictures.service
```

**Kandungan:**

```ini
[Unit]
Description=Google Drive (Pictures) Mount
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/rclone mount gdrive:Pictures %h/Pictures \
  --vfs-cache-mode writes \
  --poll-interval 1m
ExecStop=/usr/bin/fusermount3 -u %h/Pictures
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

### Servis 3: gdrive-videos.service

Akhir sekali, fail untuk `Videos`.

**Fail:** `~/.config/systemd/user/gdrive-videos.service`

```bash
nano ~/.config/systemd/user/gdrive-videos.service
```

**Kandungan:**

```ini
[Unit]
Description=Google Drive (Videos) Mount
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/rclone mount gdrive:Videos %h/Videos \
  --vfs-cache-mode writes \
  --poll-interval 1m
ExecStop=/usr/bin/fusermount3 -u %h/Videos
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

**Nota Mengenai Pilihan (Flags):**

  * `--vfs-cache-mode writes`: Menjadikan proses menulis fail lebih lancar. Aplikasi tidak akan 'beku' semasa menunggu fail dimuat naik.
  * `--poll-interval 1m`: `rclone` akan menyemak perubahan fail di Google Drive setiap 1 minit. Ini penting supaya anda sentiasa mendapat versi fail terkini.

-----

## 3\. Mengaktifkan Servis

Selepas ketiga-tiga fail disimpan, kita perlu memberitahu `systemd` tentangnya dan mengaktifkannya.

**PENTING:** Jalankan arahan ini sebagai pengguna biasa (tanpa `sudo`).

```bash
# 1. Muat semula daemon systemd user
systemctl --user daemon-reload

# 2. Aktifkan (enable) servis supaya ia bermula automatik semasa log masuk
systemctl --user enable gdrive-documents.service
systemctl --user enable gdrive-pictures.service
systemctl --user enable gdrive-videos.service

# 3. Mulakan (start) servis buat kali pertama
systemctl --user start gdrive-documents.service
systemctl --user start gdrive-pictures.service
systemctl --user start gdrive-videos.service
```

-----

## 4\. Pengesahan

Anda boleh menyemak status setiap servis untuk memastikan ia berjalan.

```bash
systemctl --user status gdrive-documents.service
systemctl --user status gdrive-pictures.service
systemctl --user status gdrive-videos.service
```

Jika semuanya berjalan lancar, anda akan melihat status `active (running)` berwarna hijau.

Anda juga boleh menyemak folder anda:

```bash
mount | grep gdrive
ls -l ~/Documents
ls -l ~/Pictures
ls -l ~/Videos
```

Anda kini sepatutnya melihat fail dan folder dari Google Drive anda terpapar di situ\!

![Konsole mount point]({{ "assets/images/konsole_mount.png" | relative_url }})

## Penutup

Dengan kaedah ini, Google Drive anda kini telah diintegrasi dengan lancar ke dalam sistem fail Linux anda. Ia tidak menggunakan ruang storan yang banyak (kerana ia *mount*, bukan *sync*) dan berjalan secara automatik di sebalik tabir. Selamat mencuba\!
