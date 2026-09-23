# Catatan Setup Arch Linux dan Snapshot Package

Repository ini mendokumentasikan workstation Arch Linux pribadi berbasis KDE Plasma. Isinya menunjukkan keputusan konfigurasi, package snapshot, dan automation Bash yang saya gunakan untuk mempelajari administrasi Linux secara praktis.

Ini bukan installer universal atau security lab. Nama user, hostname, UUID filesystem, alamat jaringan, dan path host asli sengaja tidak disimpan pada current tree agar dokumentasi aman digunakan sebagai portfolio publik.

## Tujuan

- mempertahankan catatan keputusan setup yang dapat ditinjau ulang;
- mencatat package eksplisit, package asing/AUR, dependency, dan transaksi pacman terbaru;
- menjalankan snapshot lokal setelah transaksi pacman;
- melatih penggunaan Btrfs, systemd, hybrid graphics, firewall, dan shell scripting.

## Desain Sistem

| Area | Keputusan |
| --- | --- |
| Distribusi | Arch Linux |
| Desktop | KDE Plasma minimal, Wayland-first |
| Filesystem | Btrfs dengan subvolume terpisah |
| Boot | systemd-boot, kernel reguler dan LTS |
| Graphics | AMD iGPU dan NVIDIA on-demand |
| Network | NetworkManager |
| Audio | PipeWire dan WirePlumber |
| Swap | zram |
| Firewall | UFW, deny incoming secara default |
| Maintenance | pacman hook dan snapshot Git lokal |

Pemilihan package dibuat untuk workstation harian, bukan sebagai rekomendasi tunggal untuk semua perangkat. Dukungan hardware dan kebutuhan pengguna perlu diperiksa sebelum meniru konfigurasi.

## Layout Btrfs

Struktur subvolume yang digunakan:

```text
@        -> /
@home    -> /home
@cache   -> /var/cache
@log     -> /var/log
```

Contoh mount dibuat generik agar tidak menunjuk disk milik pengguna tertentu:

```bash
mount -o noatime,compress=zstd,ssd,space_cache=v2,subvol=@ \
  /dev/<root-partition> /mnt
```

Nama device harus diverifikasi dengan `lsblk` pada mesin target. Perintah format atau partisi tidak boleh disalin tanpa backup dan pemeriksaan perangkat secara manual.

## Boot dan Kernel

Setup menggunakan kernel reguler dan LTS sebagai jalur pemulihan. Entry systemd-boot mengikuti pola berikut:

```text
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options root=UUID=<root-filesystem-uuid> rootflags=subvol=@ rw
```

Identifier filesystem pada contoh adalah placeholder. Current tree tidak menyimpan UUID atau layout disk host asli.

## Hybrid Graphics

Mesa digunakan untuk iGPU, sedangkan driver NVIDIA dan PRIME offload digunakan untuk workload yang membutuhkan GPU diskret. Konfigurasi ini mengutamakan iGPU untuk penggunaan harian dan mengaktifkan GPU diskret secara on-demand.

Verifikasi yang relevan pada host Arch:

```bash
glxinfo -B
prime-run glxinfo -B
systemctl status switcheroo-control
```

## Hardening Dasar Workstation

Kontrol yang didokumentasikan pada setup ini meliputi:

- firewall dengan kebijakan default `deny incoming` dan `allow outgoing`;
- update package melalui pacman dan pencatatan transaksi;
- pemisahan akun pengguna dari operasi root;
- script hook yang disimpan di lokasi root-owned;
- tidak ada push remote otomatis dari hook package manager.

Repository ini tidak membuktikan bahwa seluruh sistem telah di-hardening atau diaudit. Ia hanya mencatat kontrol yang benar-benar ada pada konfigurasi workstation.

## Struktur Repository

```text
README.md                         ringkasan dan keputusan konfigurasi
packages-explicit-official.txt    package resmi yang dipasang eksplisit
packages-explicit-foreign.txt     package asing/AUR yang dipasang eksplisit
packages-all.txt                  seluruh package termasuk dependency
pacman-recent.log                 transaksi pacman terbaru
scripts/update-arch-readme        pembuat laporan package tersanitasi
scripts/update-arch-snapshot      pembaruan dan commit snapshot lokal
hooks/99-update-arch-readme.hook  pacman hook PostTransaction
```

## Menyiapkan Automation

Script membaca konfigurasi lokal dari `/etc/arch-snapshot.conf`. File tersebut tidak disimpan pada repository karena memuat path dan nama akun lokal.

Contoh konfigurasi:

```bash
SNAPSHOT_DIR="/path/to/local/clone"
SNAPSHOT_OWNER="<local-user>"
SNAPSHOT_BRANCH="main"
```

Instalasi pada host Arch dilakukan secara eksplisit oleh administrator:

```bash
sudo install -m 0755 scripts/update-arch-readme /usr/local/bin/update-arch-readme
sudo install -m 0755 scripts/update-arch-snapshot /usr/local/bin/update-arch-snapshot
sudo install -m 0644 hooks/99-update-arch-readme.hook \
  /etc/pacman.d/hooks/99-update-arch-readme.hook
sudoedit /etc/arch-snapshot.conf
sudo chmod 0600 /etc/arch-snapshot.conf
```

Setelah konfigurasi diisi, syntax dan pembuatan snapshot dapat diperiksa dengan:

```bash
bash -n scripts/update-arch-readme
bash -n scripts/update-arch-snapshot
sudo /usr/local/bin/update-arch-readme
```

`update-arch-snapshot` membuat commit lokal hanya pada branch yang dikonfigurasi dan hanya men-stage file snapshot yang sudah ditentukan. Script tidak melakukan `git push`, force push, orphan checkout, penghapusan history, atau garbage collection agresif. Publikasi dilakukan manual setelah diff diperiksa.

## Package Snapshot

File package adalah snapshot satu workstation pada waktu tertentu, bukan daftar package yang wajib dipasang. Daftar eksplisit lebih berguna untuk merekonstruksi intent, sedangkan `packages-all.txt` membantu menelusuri dependency yang terpasang.

<!-- AUTO_PACKAGE_REPORT_START -->
Snapshot dibuat pada `2026-09-23T13:48:26+07:00`.

| Jenis | Jumlah |
| --- | ---: |
| Package resmi eksplisit | 157 |
| Package asing/AUR eksplisit | 0 |
| Seluruh package | 1214 |

Kernel release: `7.2.6-arch2-1`.

Daftar lengkap tersedia pada file `packages-*.txt`; transaksi terbaru tersedia pada `pacman-recent.log`. Laporan tidak memasukkan hostname, username, UUID, disk layout, atau mount path.
<!-- AUTO_PACKAGE_REPORT_END -->

## Pengujian

Tidak ada test framework karena project ini berupa dokumentasi dan script administrasi. Validasi yang tersedia adalah:

- `bash -n` untuk syntax kedua script;
- `shellcheck` bila tool tersedia;
- pemeriksaan bahwa marker laporan otomatis tetap seimbang;
- review `git diff` sebelum commit atau publikasi.

Eksekusi penuh hook belum dapat dianggap portable: ia memerlukan Arch Linux, pacman, systemd, akses root, repository lokal, dan konfigurasi `/etc/arch-snapshot.conf` yang valid.

## Catatan Keamanan

- jangan commit `/etc/arch-snapshot.conf`, credential Git, token, atau remote URL berisi credential;
- review package list karena daftar software dapat menambah informasi reconnaissance;
- jangan publikasikan hostname, username lokal, UUID, alamat IP, mount pribadi, atau output `uname -a`;
- simpan script dan hook terpasang sebagai milik `root` agar transaksi pacman tidak menjalankan file yang dapat diubah pengguna biasa;
- push remote harus tetap menjadi tindakan manual setelah review.

## Yang Saya Pelajari

Project ini mencatat pembelajaran tentang pemilihan package minimal, boot dan recovery kernel, subvolume Btrfs, hybrid graphics, service management, firewall dasar, serta automation yang membatasi scope perubahan Git.

## Status

Snapshot dokumentasi workstation pribadi. Current tree telah digeneralisasi untuk portfolio; detail lama masih dapat berada pada Git history dan perlu ditinjau sebelum repository private ini diubah menjadi public.
