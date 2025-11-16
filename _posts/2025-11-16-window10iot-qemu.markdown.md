---
layout: post
title: "Panduan Sandbox Windows 10 IoT LTSC Guna KVM/QEMU di Arch Linux"
date: 2025-11-16 16:55:00 +0800
categories: [tech, linux, virtualisasi]
tags: [archlinux, kvm, qemu, virt-manager, windows 10, ltsc, panduan, virtualisasi, software testing, sysadmin, putrajaya]
---

Walaupun sokongan rasmi untuk Windows 10 versi pengguna telah berakhir, ia tidak bermakna platform itu telah hilang dari persekitaran produksi.

Banyak organisasi, termasuk Perbadanan Putrajaya dan lain-lain dalam sektor industri dan korporat, masih bergantung penuh pada **Windows 10 IoT LTSC (Long-Term Servicing Channel)**. Versi ini mempunyai kitaran sokongan yang masih panjang, bermakna kita sebagai pentadbir sistem masih perlu menyokongnya untuk beberapa tahun lagi.

Ini mewujudkan keperluan untuk persekitaran ujian (*sandbox*) yang stabil. Kita perlu menguji keserasian aplikasi, *patch* keselamatan, dan *group policy* sebelum melaksanakannya di mesin produksi LTSC.

Sebagai pengguna Arch Linux untuk kerja harian, saya mendapati KVM/QEMU adalah kaedah yang paling mudah untuk mewujudkan *lab* ujian ini. Ia adalah hipervisor *native* Linux dan menawarkan prestasi I/O yang pantas menggunakan VirtIO.

Panduan ini adalah dokumentasi langkah-langkah saya menyediakan *guest* Windows 10 LTSC di atas *host* Arch Linux menggunakan `virt-manager`.

---

### 🖥️ Langkah 1: Pemasangan Pakej KVM/QEMU

Pertama, pastikan CPU anda menyokong virtualisasi (VT-x untuk Intel atau AMD-V untuk AMD) dan ia telah diaktifkan di dalam BIOS/UEFI.

1.  **Sahkan Sokongan Virtualisasi:**
    Jalankan arahan ini di terminal anda:
    ```bash
    lscpu | grep Virtualization
    ```
    
    **Memahami Output:**
    * **`VT-x`**: Menandakan CPU Intel anda menyokong Intel Virtualization Technology.
    * **`AMD-V`**: Menandakan CPU AMD anda menyokong AMD Virtualization Technology.
    * **Jika tiada output:** CPU anda mungkin tidak menyokong virtualisasi perkakasan, atau ia mungkin telah dilumpuhkan (disabled) di dalam tetapan BIOS/UEFI anda.

2.  **Pasang Pakej:**
    Setelah virtualisasi disahkan, pasang semua pakej yang diperlukan dari repositori rasmi Arch.
    ```bash
    sudo pacman -S qemu-desktop libvirt virt-manager edk2-ovmf dnsmasq iptables-nft
    ```
    * `qemu-desktop`: Pakej QEMU dan alatan yang berkaitan.
    * `libvirt`: Daemon pengurusan latar belakang untuk VM.
    * `virt-manager`: Aplikasi GUI untuk mengurus VM anda.
    * `edk2-ovmf`: Menyediakan perisian tegar (firmware) UEFI moden untuk VM.
    * `dnsmasq` & `iptables-nft`: Diperlukan untuk rangkaian NAT lalai (akses internet untuk VM).



### ⚙️ Langkah 2: Konfigurasi Servis dan Kebenaran (Host)

Seterusnya, kita perlu memulakan servis `libvirt` dan memberi kebenaran kepada pengguna anda untuk mengurus VM tanpa `sudo`.

1.  **Mulakan dan Aktifkan Servis `libvirtd`:**
    ```bash
    sudo systemctl enable --now libvirtd.service
    ```

2.  **Tambah Pengguna Anda ke Kumpulan `libvirt`:**
    ```bash
    sudo usermod -aG libvirt $(whoami)
    ```
    > **PENTING:** Anda mesti **log keluar dan log masuk semula** (atau *reboot*) supaya keahlian kumpulan baru ini berkuat kuasa.

3.  **Mulakan Rangkaian Lalai:**
    Ini akan menyediakan rangkaian NAT "default" untuk VM anda.
    ```bash
    sudo virsh net-start default
    sudo virsh net-autostart default
    ```



### 🚀 Langkah 3: Mencipta VM Windows 10

Kini kita akan mencipta mesin maya itu sendiri.

1.  Buka `virt-manager` dari menu aplikasi anda.
2.  Klik ikon "Create a new virtual machine".
3.  Pilih **"Local install media"** dan cari fail `.iso` Windows 10 LTSC anda. `virt-manager` selalunya akan mengesan OS secara automatik.
4.  Peruntukkan jumlah RAM dan bilangan CPU yang anda inginkan untuk *lab* ujian anda.
5.  Cipta imej cakera maya (format `qcow2`) untuk VM anda. Saiz 60GB-80GB adalah permulaan yang baik.
6.  Pada langkah terakhir, **tandakan (tick)** kotak **"Customize configuration before install"** dan klik **Finish**.
[Imej tetingkap 'Customize configuration before install' dalam virt-manager]

7.  **Dalam tetingkap kustomisasi (PENTING UNTUK PRESTASI):**
    * **Overview**: Pastikan **Firmware** ditetapkan kepada **UEFI**.
    * **SATA Disk 1**: Pergi ke "Advanced options". Tukar **"Disk bus"** dari "SATA" kepada **"VirtIO"**. Ini memberikan prestasi I/O cakera yang jauh lebih pantas, kritikal untuk *testing*.
    * **NIC (Network Card)**: Pergi ke "Device model" dan tukar kepada **"virtio"** untuk prestasi rangkaian yang optimum.

8.  Klik **"Begin Installation"** di penjuru atas kiri untuk memulakan VM dan proses pemasangan.



### 💿 Langkah 4: Memuatkan Pemacu (Driver) VirtIO Semasa Pemasangan

Apabila anda memulakan pemasangan Windows 10, anda akan tiba di skrin pemilihan cakera, tetapi ia akan kelihatan kosong.

[Imej skrin pemasangan Windows yang menunjukkan 'No drives found']

Ini adalah normal. Windows tidak mempunyai pemacu (driver) untuk pengawal cakera "VirtIO" yang pantas itu.

1.  **Muat Turun ISO VirtIO:**
    Gunakan hos Arch anda untuk memuat turun fail ISO pemacu VirtIO-Win terkini dari [Fedora VirtIO-Win project](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso). Simpan fail `virtio-win.iso` ini.

2.  **Tambah ISO ke VM:**
    * Dalam `virt-manager`, pergi ke paparan "Details" VM anda (ikon mentol).
    * Klik "Add Hardware" > "Storage".
    * Tetapkan "Device type" kepada "CDROM device".
    * Klik "Manage" dan pilih fail `virtio-win.iso` yang anda muat turun tadi.

3.  **Semasa Pemasangan Windows (di skrin yang kosong tadi):**
    * Klik **"Load driver"**.
    * Klik "Browse".
    * Navigasi ke CD-ROM VirtIO (biasanya `D:` atau `E:`).
    * Pilih folder: `viostor/w10/amd64`
    * Klik "OK". Ia akan memuatkan "Red Hat VirtIO SCSI controller".
    * Klik "Next".
[Imej tetingkap Load Driver Windows 10 menunjukkan folder viostor/w10/amd64]

4.  Cakera maya anda (yang menggunakan VirtIO) kini akan muncul. Teruskan pemasangan Windows seperti biasa.

5.  **Tugasan Pasca-Pemasangan:**
    Setelah anda log masuk ke Windows 10, buka File Explorer. Pergi ke CD-ROM VirtIO tadi dan jalankan fail `virtio-win-guest-tools.exe`. Ini akan memasang semua pemacu yang lain (rangkaian, paparan, QEMU guest agent) untuk prestasi maksimum dan integrasi yang lebih baik (seperti saiz skrin dinamik).



### 🎉 Kesimpulan

Tahniah! Anda kini mempunyai persekitaran ujian Windows 10 LTSC yang stabil dan pantas di atas Arch Linux. Dengan pemacu VirtIO, prestasi VM adalah sangat optimum untuk sebarang tugasan pengujian perisian (*software testing*), *debugging*, atau pembangunan.

Konfigurasi ini membolehkan anda mengasingkan persekitaran ujian anda dengan selamat tanpa menjejaskan sistem *host* utama anda.

Walaupun panduan ini menggunakan Arch Linux, kaedah yang sama boleh digunapakai pada distro Linux yang lain (seperti Fedora, Ubuntu, atau Debian). Anda hanya perlu menyesuaikan arahan pengurus pakej (contohnya `apt` atau `dnf` dan bukannya `pacman`) untuk memasang `qemu`, `libvirt`, dan `virt-manager`.

![](/assets/images/win10iot_qemu.png)