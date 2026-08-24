# Arch Linux KDE Plasma Clean Setup — Nitro ANV15-42

Dokumentasi instalasi Arch Linux KDE Plasma clean, ringan, stabil, dan daily-use pada laptop Acer Nitro ANV15-42 dual boot dengan Windows.

Dokumentasi ini dibuat supaya seluruh proses instalasi, tech stack, konfigurasi, dan perubahan package bisa dilacak tanpa harus mengingat ulang dari terminal.

---

## 1. Target Sistem

Setup ini dirancang untuk:

* Arch Linux clean.
* KDE Plasma minimal, bukan full KDE bloat.
* Wayland-first.
* Daily driver stabil.
* Dual boot aman dengan Windows.
* AMD iGPU untuk penggunaan harian.
* NVIDIA RTX 4050 aktif secara on-demand.
* Audio modern dengan PipeWire.
* Swap ringan dengan zram.
* Boot aman dengan systemd-boot.
* Dokumentasi package otomatis setelah transaksi pacman.

---

## 2. Hardware

```txt
Laptop        : Acer Nitro ANV15-42
CPU           : AMD Ryzen 7 7445HS
GPU           : NVIDIA GeForce RTX 4050 Laptop GPU 6GB
iGPU          : AMD/ATI HawkPoint
RAM           : 16GB DDR5
Display       : 15.6" 1920x1080 180Hz
Storage utama : NVMe 512GB
Wi-Fi         : IEEE 802.11ax
Bluetooth     : Bluetooth 5.3 atau lebih tinggi
BIOS version  : 01.09
BIOS date     : 08/12/2025
```

---

## 3. Keputusan Desain

| Area            | Keputusan                              |
| --------------- | -------------------------------------- |
| Distro          | Arch Linux                             |
| Desktop         | KDE Plasma minimal                     |
| Display Manager | SDDM                                   |
| Session         | Wayland-first                          |
| Bootloader      | systemd-boot                           |
| Dual Boot       | Windows + Arch                         |
| Filesystem Arch | Btrfs                                  |
| Btrfs Subvolume | `@`, `@home`, `@cache`, `@log`         |
| Kernel          | `linux` + `linux-lts`                  |
| CPU Microcode   | `amd-ucode`                            |
| GPU             | AMD Mesa + NVIDIA open driver          |
| NVIDIA Offload  | `nvidia-prime` + `switcheroo-control`  |
| Network         | NetworkManager + `wpa_supplicant`      |
| Bluetooth       | BlueZ + Bluedevil                      |
| Audio           | PipeWire + WirePlumber                 |
| Power           | PowerDevil + power-profiles-daemon     |
| Swap            | zram-generator                         |
| Firewall        | ufw                                    |
| Browser         | Firefox                                |
| Boot partition  | Arch `/boot` sendiri 1GB               |
| EFI Windows     | Dipakai sebagai `/efi`, tidak diformat |

---

## 4. Package Stack Awal yang Dikunci

Package stack awal sengaja dipilih manual satu-satu. Tidak memakai `plasma`, `plasma-meta`, atau `kde-applications`.

```txt
base
linux
linux-firmware
amd-ucode
sudo
nano
vim
man-db
man-pages
pacman-contrib
reflector
git
bash-completion

linux-lts
linux-headers
linux-lts-headers

efibootmgr
dosfstools
e2fsprogs
btrfs-progs
ntfs-3g

networkmanager
plasma-nm
wpa_supplicant
wireless-regdb

bluez
bluez-utils
bluedevil

pipewire
pipewire-audio
pipewire-pulse
pipewire-alsa
wireplumber
alsa-utils
sof-firmware
plasma-pa

mesa
vulkan-radeon
libva-mesa-driver

nvidia-open
nvidia-open-lts
nvidia-utils
nvidia-settings
nvidia-prime
switcheroo-control

plasma-desktop
sddm
konsole
dolphin
systemsettings
kde-cli-tools
kscreen
powerdevil
breeze
xdg-desktop-portal
xdg-desktop-portal-kde
xdg-utils
kwallet-pam

kio-extras
ark
spectacle
okular
gwenview
kate
ffmpegthumbs

zip
unzip
p7zip
unrar

power-profiles-daemon
upower
zram-generator

libinput

noto-fonts
noto-fonts-emoji
ttf-dejavu
ttf-liberation
ttf-jetbrains-mono

firefox
ufw
fwupd
```

---

## 5. Koreksi Package Saat Instalasi

Saat proses `pacstrap`, ada beberapa koreksi nama package.

```txt
wireless_regdb        -> wireless-regdb
uzip                  -> unzip
power-profile-daemon  -> power-profiles-daemon
fwupdfwupd            -> fwupd
```

Package ini dihapus dari stack awal karena tidak tersedia di repo Arch saat instalasi:

```txt
mesa-vdpau
```

Package audio tambahan setelah validasi:

```txt
alsa-card-profiles
rtkit
```

Catatan:

```txt
alsa-card-profiles sudah tersedia/reinstalled.
rtkit ditambahkan agar integrasi realtime audio PipeWire lebih baik.
```

---

## 6. Package / Group yang Sengaja Tidak Dipasang

Package berikut sengaja tidak dipasang agar sistem tetap clean:

```txt
plasma
plasma-meta
kde-applications
discover
packagekit-qt6
iwd
grub
tlp
noto-fonts-cjk
xf86-input-libinput
AUR helper
```

Alasan utama:

* `plasma` dan `kde-applications` terlalu besar.
* `discover` dan `packagekit-qt6` tidak dibutuhkan di setup pacman manual.
* `iwd` tidak dipakai karena NetworkManager + wpa_supplicant lebih stabil untuk awal.
* `grub` tidak dipakai karena systemd-boot lebih clean.
* `tlp` tidak dipakai karena kita memakai PowerDevil + power-profiles-daemon.
* AUR helper ditunda sampai sistem dasar stabil.

---

## 7. Persiapan Windows Sebelum Install

Tahap Windows yang dilakukan:

```txt
[x] Backup data penting
[x] Cek BIOS Mode = UEFI
[x] Cek BitLocker / Device Encryption
[x] BitLocker OFF
[x] Device Encryption OFF
[x] Hibernate/Fast Startup dimatikan
[x] Windows Update aman
[x] Shrink / siapkan ruang kosong
[x] Windows tetap login normal setelah instalasi
```

Command Windows yang dijalankan:

```cmd
powercfg.exe /hibernate off
manage-bde -status
```

Hasil BitLocker:

```txt
Volume C:
BitLocker Version: None
Conversion Status: Fully Decrypted
Percentage Encrypted: 0.0%
Encryption Method: None
Protection Status: Protection Off
Lock Status: Unlocked
Key Protectors: None Found
```

---

## 8. Layout Disk Akhir

Layout disk utama:

```txt
/dev/nvme0n1      476.9G  NVMe utama
├─nvme0n1p1       200M    EFI Windows
├─nvme0n1p2        16M    Microsoft Reserved
├─nvme0n1p3      99.2G    Windows C:
├─nvme0n1p4       780M    Windows Recovery
├─nvme0n1p5         1G    ARCHBOOT
└─nvme0n1p6     375.7G    ARCHROOT
```

Mount final:

```txt
/efi          -> /dev/nvme0n1p1
/boot         -> /dev/nvme0n1p5
/             -> /dev/nvme0n1p6 subvol=@
/home         -> /dev/nvme0n1p6 subvol=@home
/var/cache    -> /dev/nvme0n1p6 subvol=@cache
/var/log      -> /dev/nvme0n1p6 subvol=@log
```

Alasan membuat `/boot` sendiri:

```txt
EFI Windows hanya 200MB.
Arch memakai dua kernel: linux dan linux-lts.
Agar tidak memenuhi EFI Windows, dibuat partisi /boot sendiri 1GB.
```

---

## 9. Tahapan Instalasi yang Dilakukan

### 9.1 Live ISO

```txt
[x] Boot Arch ISO
[x] Internet aktif
[x] timedatectl NTP aktif
[x] Mirror Singapore dipakai
[x] pacman -Syy sukses
[x] UEFI mode aman
[x] Disk NVMe terbaca
[x] GPU AMD + NVIDIA RTX 4050 terbaca
```

Catatan mirror:

```txt
reflector sempat gagal dengan error no mirrors found.
Solusi: mirrorlist Singapore dibuat manual dari mirrorlist resmi / fallback.
```

---

### 9.2 Partisi

Partisi Arch dibuat dari ruang kosong:

```txt
/dev/nvme0n1p5  1G       Linux extended boot / ARCHBOOT
/dev/nvme0n1p6  375.7G   Linux filesystem / ARCHROOT
```

Windows tidak disentuh:

```txt
/dev/nvme0n1p1
/dev/nvme0n1p2
/dev/nvme0n1p3
/dev/nvme0n1p4
```

---

### 9.3 Format

```bash
mkfs.fat -F32 -n ARCHBOOT /dev/nvme0n1p5
mkfs.btrfs -f -L ARCHROOT /dev/nvme0n1p6
```

---

### 9.4 Btrfs Subvolume

```bash
mount /dev/nvme0n1p6 /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@log

umount /mnt
```

---

### 9.5 Mount Final

```bash
mount -o noatime,compress=zstd,ssd,space_cache=v2,subvol=@ /dev/nvme0n1p6 /mnt

mkdir -p /mnt/{home,var/cache,var/log,boot,efi}

mount -o noatime,compress=zstd,ssd,space_cache=v2,subvol=@home /dev/nvme0n1p6 /mnt/home
mount -o noatime,compress=zstd,ssd,space_cache=v2,subvol=@cache /dev/nvme0n1p6 /mnt/var/cache
mount -o noatime,compress=zstd,ssd,space_cache=v2,subvol=@log /dev/nvme0n1p6 /mnt/var/log

mount /dev/nvme0n1p5 /mnt/boot
mount /dev/nvme0n1p1 /mnt/efi
```

---

### 9.6 Install Package

Package diinstall memakai `pacstrap -K /mnt`.

Catatan:

```txt
Pacstrap pertama gagal karena typo nama package.
Pacstrap kedua sukses setelah package dikoreksi.
```

---

### 9.7 Fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

Hasil validasi:

```txt
[x] / masuk fstab sebagai Btrfs subvol=@
[x] /home masuk fstab sebagai subvol=@home
[x] /var/cache masuk fstab sebagai subvol=@cache
[x] /var/log masuk fstab sebagai subvol=@log
[x] /boot masuk fstab
[x] /efi masuk fstab
```

---

### 9.8 Chroot dan Basic Config

```bash
arch-chroot /mnt
```

Konfigurasi dasar:

```txt
Timezone   : Asia/Jakarta
Locale     : en_US.UTF-8 dan id_ID.UTF-8
User       : doni-arch
Hostname   : archlinux
Sudo       : wheel enabled
```

---

### 9.9 Initramfs dan NVIDIA

NVIDIA modules dimasukkan ke initramfs:

```txt
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

Config NVIDIA modeset:

```txt
/etc/modprobe.d/nvidia.conf
options nvidia_drm modeset=1
```

Initramfs digenerate:

```bash
mkinitcpio -P
```

Hasil:

```txt
[x] linux initramfs sukses
[x] linux-lts initramfs sukses
```

Warning `sd-vconsole` muncul karena `/etc/vconsole.conf` tidak ada. Ini aman karena keyboard default US.

---

### 9.10 Zram

Config:

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
```

Hasil validasi:

```txt
/dev/zram0 aktif
Size sekitar 7.5G
Priority 100
Algorithm zstd
```

---

### 9.11 Service yang Diaktifkan

```bash
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable sddm
systemctl enable power-profiles-daemon
systemctl enable switcheroo-control
systemctl enable ufw
systemctl enable fstrim.timer
systemctl enable systemd-timesyncd
```

Firewall:

```bash
ufw default deny incoming
ufw default allow outgoing
ufw --force enable
```

---

### 9.12 Bootloader systemd-boot

Layout:

```txt
/efi   -> EFI Windows
/boot  -> ARCHBOOT
```

Install systemd-boot:

```bash
bootctl --esp-path=/efi --boot-path=/boot install
```

Manual EFI entry dibuat karena bootctl warning soal EFI variable:

```bash
efibootmgr \
  --create \
  --disk /dev/nvme0n1 \
  --part 1 \
  --label "Arch Linux" \
  --loader '\EFI\systemd\systemd-bootx64.efi'
```

Boot order akhir:

```txt
Boot0002* Arch Linux
Boot0001* Windows Boot Manager
BootOrder: 0002,0001,2001,2002,2003
```

---

## 10. Boot Entry

### 10.1 Loader Config

File:

```txt
/efi/loader/loader.conf
```

Isi:

```ini
default arch.conf
timeout 4
console-mode max
editor no
```

---

### 10.2 Arch Linux

File:

```txt
/boot/loader/entries/arch.conf
```

Isi:

```ini
title   Arch Linux by Pria-Solo
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options root=UUID=4026ccf1-6782-444a-a879-ec3325eec1c5 rootflags=subvol=@ rw nvidia_drm.modeset=1
```

---

### 10.3 Arch Linux LTS

File:

```txt
/boot/loader/entries/arch-lts.conf
```

Isi:

```ini
title   Arch Linux LTS by Pria-Solo
linux   /vmlinuz-linux-lts
initrd  /amd-ucode.img
initrd  /initramfs-linux-lts.img
options root=UUID=4026ccf1-6782-444a-a879-ec3325eec1c5 rootflags=subvol=@ rw nvidia_drm.modeset=1
```

---

## 11. First Boot

Hasil first boot:

```txt
[x] systemd-boot berhasil
[x] Arch Linux berhasil boot
[x] SDDM muncul
[x] Plasma Wayland tersedia
[x] Login user awal sempat bermasalah karena lupa user/password
[x] Root TTY berhasil dipakai untuk memperbaiki user
[x] Login KDE akhirnya aman
```

---

## 12. Validasi Sistem

### 12.1 Session

```bash
echo $XDG_SESSION_TYPE
```

Hasil:

```txt
wayland
```

---

### 12.2 Sudo

```bash
sudo whoami
```

Hasil:

```txt
root
```

---

### 12.3 Service

```bash
systemctl --failed
```

Hasil:

```txt
0 loaded units listed
```

Service aktif:

```txt
NetworkManager
bluetooth
sddm
power-profiles-daemon
switcheroo-control
fstrim.timer
```

---

### 12.4 Network

```bash
nmcli device status
```

Hasil:

```txt
wlp4s0 connected Yellow Kost 2,4G
```

Ping:

```txt
archlinux.org reachable
0% packet loss
```

---

### 12.5 NVIDIA

```bash
nvidia-smi
```

Hasil:

```txt
NVIDIA GeForce RTX 4050 Laptop GPU terdeteksi
Driver NVIDIA aktif
```

Modules:

```txt
nvidia
nvidia_drm
nvidia_modeset
nvidia_uvm
```

---

### 12.6 Zram

```bash
swapon --show
zramctl
```

Hasil:

```txt
/dev/zram0 aktif
Size sekitar 7.5G
Priority 100
Algorithm zstd
```

---

### 12.7 Audio

PipeWire:

```txt
pipewire active running
wireplumber active running
```

Audio devices:

```txt
Speaker: Ryzen HD Audio Controller Speaker
Microphone: Ryzen HD Audio Controller Digital Microphone
```

Video:

```txt
Webcam: ACER HD User Facing
```

Catatan:

```txt
Warning libcamera SPA plugin missing diabaikan dulu karena webcam tetap terdeteksi via v4l2.
Warning api.alsa.acp.device tidak fatal karena speaker dan mic tetap muncul.
```

---

### 12.8 Power Profile

```bash
powerprofilesctl
```

Hasil:

```txt
performance
balanced
power-saver
```

Default:

```txt
balanced
```

Driver:

```txt
CpuDriver: amd_pstate
PlatformDriver: platform_profile
```

---

## 13. Masalah BIOS / Firmware

Masalah yang ditemukan:

```txt
BIOS bisa dibuka.
Saat klik Exit Saving Changes -> Yes, BIOS kembali ke menu Exit Saving Changes lalu freeze/stuck.
```

Status:

```txt
Arch tetap boot aman.
Windows tetap boot aman.
Windows login aman.
EFI entry Arch dan Windows aman.
Masalah dianggap firmware/BIOS setup issue, bukan masalah Arch.
```

Saran sementara:

```txt
Jangan masuk BIOS kalau tidak perlu.
Jika hanya ingin pilih boot device, pakai F12 Boot Menu.
Jika masuk BIOS dan tidak mengubah apa-apa, jangan pakai Exit Saving Changes.
Gunakan Exit Discarding Changes jika tersedia.
```

Rencana lanjutan jika masalah BIOS tetap muncul:

```txt
Cek update BIOS resmi Acer dari Windows.
Update hanya jika versi lebih baru tersedia untuk Nitro ANV15-42.
Pastikan charger terpasang dan file BIOS cocok dengan model/SNID.
```

---

## 14. Rekomendasi Setting KDE Awal

Setting ringan dan nyaman:

```txt
Theme: Breeze / Breeze Dark
Session: Wayland
Animation speed: fast sekitar 70–80%
File Search: ON
Content indexing: OFF
Startup session: Empty session
Power profile: Balanced
Refresh rate: 180Hz saat charger
Touchpad tap-to-click: ON
Screen lock: 10–15 menit
Autostart: minimal
NVIDIA: on-demand saja
```

Jangan utak-atik dulu:

```txt
Compositor advanced
NVIDIA config manual
SDDM theme luar
Global theme dari KDE Store
AUR helper
TLP
```

---

## 15. SSD Kedua / Data Drive

Rencana untuk SSD kedua yang masih unallocated:

```txt
Gunakan sebagai shared DATA drive.
Format dari Windows sebagai NTFS.
Label: DATA.
Drive letter: D: atau E:.
```

Alasan:

```txt
Windows bisa baca/tulis.
Arch bisa baca/tulis karena ntfs-3g sudah tersedia.
Tidak mengganggu root Arch.
Tidak perlu swap tambahan karena zram sudah aktif.
```

---

## 16. Command Maintenance

Update sistem:

```bash
sudo pacman -Syu
```

Cek failed service:

```bash
systemctl --failed
```

Cek orphan package:

```bash
pacman -Qtdq
```

Cek NVIDIA:

file:///home/doni-arch/Pictures/pngegg.png```bash
nvidia-smi
```

Cek zram:

```bash
swapon --show
zramctl
```

Cek audio:

```bash
wpctl status
```

Cek boot entry:

```bash
efibootmgr -v
```

Cek power profile:

```bash
powerprofilesctl
```

Cek filesystem:

```bash
findmnt -R /
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS
```

---

## 17. Auto Package Report

Bagian di bawah ini akan diperbarui otomatis setelah transaksi pacman.

<!-- AUTO_PACKAGE_REPORT_START -->
## Auto Package Report

Generated automatically after pacman transaction.

### Last Updated

```txt
Mon Aug 24 12:13:49 PM WIB 2026
```

### Package Count

| Type | Count |
|---|---:|
| Explicit official packages | 154 |
| Explicit foreign/AUR packages | 0 |
| All installed packages including dependencies | 1195 |

### Explicit Official Packages

```txt
7zip
alsa-utils
amd-ucode
android-tools
ark
base
base-devel
bash-completion
bat
bc
bind
bluedevil
bluez
bluez-utils
breeze
btop
btrfs-progs
clang
cmake
discord
docker
docker-buildx
docker-compose
dolphin
dosfstools
e2fsprogs
efibootmgr
ethtool
eza
fastfetch
fd
ffmpegthumbs
firefox
fwupd
fzf
gamemode
gdb
ghidra
git
github-cli
go
gradle
gvim
gwenview
hexedit
htop
inetutils
jdk17-openjdk
jdk-openjdk
jq
kate
kde-cli-tools
kio-extras
konsole
kotlin
kscreen
kwallet-pam
lib32-gamemode
lib32-mangohud
lib32-mesa
lib32-nvidia-utils
lib32-vulkan-radeon
libinput
libreoffice-fresh
linux
linux-firmware
linux-headers
linux-lts
linux-lts-headers
lldb
ltrace
man-db
mangohud
man-pages
maven
mesa
mesa-utils
nano
net-tools
networkmanager
ninja
nmap
nodejs
noto-fonts
noto-fonts-emoji
npm
ntfs-3g
nvidia-open
nvidia-open-lts
nvidia-prime
nvidia-settings
nvidia-utils
nvme-cli
okular
openbsd-netcat
openssh
pacman-contrib
perl-archive-zip
perl-image-exiftool
pipewire
pipewire-alsa
pipewire-audio
pipewire-pulse
plasma-desktop
plasma-nm
plasma-pa
powerdevil
power-profiles-daemon
qt5-declarative
reflector
rsync
rtkit
ruby
sddm
shellcheck
shfmt
smartmontools
sof-firmware
spectacle
steam
strace
sudo
switcheroo-control
systemsettings
tailscale
tcpdump
tesseract-data-eng
tesseract-data-ind
tldr
tmux
tree
ttf-dejavu
ttf-jetbrains-mono
ttf-liberation
ufw
unrar
unzip
upower
usbutils
valgrind
vlc
vlc-plugins-all
vulkan-radeon
vulkan-tools
wget
wireless-regdb
wireplumber
wireshark-qt
wpa_supplicant
xdg-desktop-portal
xdg-desktop-portal-kde
xdg-utils
zip
zram-generator
```

### Explicit Foreign / AUR Packages

```txt
```

### Recent Pacman Transactions

```txt
[2026-08-21T07:23:16+0700] [ALPM] upgraded qt6-svg (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded kiconthemes (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded kitemviews (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded knotifications (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded kjobwidgets (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded kservice (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded kwindowsystem (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded qt6-shadertools (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded qt6-5compat (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded qca-qt6 (2.3.10-7 -> 2.3.10-8)
[2026-08-21T07:23:16+0700] [ALPM] upgraded kwallet (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded solid (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded kio (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:16+0700] [ALPM] upgraded baloo (6.28.0-1 -> 6.29.0-2)
[2026-08-21T07:23:16+0700] [ALPM] upgraded bind (9.20.26-1 -> 9.20.27-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded bluez-qt (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded boost-libs (1.91.0-2 -> 1.92.0-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded cfitsio (1:4.6.4-1 -> 1:4.7.0-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded confuse (3.3-5 -> 3.4-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded containerd (2.3.3-1 -> 2.3.4-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded debuginfod (0.195-8 -> 0.196-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded discord (1:1.0.152-1 -> 1:1.0.154-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded docker-buildx (0.36.0-1 -> 0.36.1-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded docker-compose (5.4.0-1 -> 5.5.0-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded elfutils (0.195-8 -> 0.196-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded fastfetch (2.67.0-1 -> 2.67.1-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded firefox (153.0.4-1 -> 154.0-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded fluidsynth (2.5.7-1 -> 2.6.0-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded kconfigwidgets (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded kirigami (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:17+0700] [ALPM] upgraded kglobalaccel (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:18+0700] [ALPM] upgraded kxmlgui (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:18+0700] [ALPM] upgraded kcmutils (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:18+0700] [ALPM] upgraded kpackage (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:18+0700] [ALPM] upgraded syndication (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:18+0700] [ALPM] upgraded knewstuff (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:18+0700] [ALPM] upgraded frameworkintegration (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:18+0700] [ALPM] upgraded fuse2 (2.9.9-5 -> 2.9.9-6)
[2026-08-21T07:23:18+0700] [ALPM] upgraded fzf (0.74.2-1 -> 0.74.3-1)
[2026-08-21T07:23:18+0700] [ALPM] upgraded jdk-openjdk (26.0.2.u10-1 -> 26.0.2.1.u1-1)
[2026-08-21T07:23:19+0700] [ALPM] upgraded jdk21-openjdk (21.0.12.u8-1 -> 21.0.12.1.u1-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded gradle (9.6.1-1 -> 9.7.0-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded harfbuzz-icu (14.3.0-1 -> 14.3.1-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded htop (3.5.2-1 -> 3.5.3-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded imath (3.2.2-6 -> 3.2.3-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded iproute2 (7.1.0-1 -> 7.2.0-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded jdk17-openjdk (17.0.20.u8-1 -> 17.0.20.1.u1-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded kauth (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded kdeclarative (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded kded (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded kpty (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded kdesu (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:20+0700] [ALPM] upgraded kdnssd (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kholidays (1:6.28.0-1 -> 1:6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kimageformats (6.28.1-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kitemmodels (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded knotifyconfig (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kparts (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kquickcharts (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded krunner (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kstatusnotifieritem (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded ksvg (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded qt6-multimedia-ffmpeg (6.11.1-2 -> 6.11.2-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded qt6-multimedia (6.11.1-2 -> 6.11.2-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded qt6-speech (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded sonnet (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded syntax-highlighting (6.28.1-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded ktexteditor (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded ktextwidgets (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kunitconversion (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kuserfeedback (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded qt6-tools (6.11.1-4 -> 6.11.2-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded qt6-positioning (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded layer-shell-qt (6.7.4-1 -> 6.7.4-2)
[2026-08-21T07:23:21+0700] [ALPM] upgraded kwin (6.7.4-3 -> 6.7.4-7)
[2026-08-21T07:23:21+0700] [ALPM] upgraded ldb (2:4.24.5-1 -> 2:4.24.6-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded lib32-libelf (0.195-1 -> 0.196-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded lib32-libice (1.1.1-2 -> 1.1.2-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded lib32-libxfixes (6.0.1-2 -> 6.0.2-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded lib32-libxxf86vm (1.1.5-2 -> 1.1.7-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded lib32-mesa (1:26.1.6-1 -> 1:26.1.8-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded lib32-nss (3.126-1 -> 3.127-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded vulkan-mesa-implicit-layers (1:26.1.6-1 -> 1:26.1.8-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded lib32-vulkan-mesa-implicit-layers (1:26.1.6-1 -> 1:26.1.8-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded vulkan-radeon (1:26.1.6-1 -> 1:26.1.8-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded lib32-vulkan-radeon (1:26.1.6-1 -> 1:26.1.8-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded libcmis (0.6.3-1 -> 0.6.3-2)
[2026-08-21T07:23:21+0700] [ALPM] upgraded libgit2 (1:1.9.6-1 -> 1:1.9.7-1)
[2026-08-21T07:23:21+0700] [ALPM] upgraded libixion (0.20.0-7 -> 0.20.0-8)
[2026-08-21T07:23:21+0700] [ALPM] upgraded liborcus (0.21.0-6 -> 0.21.0-7)
[2026-08-21T07:23:23+0700] [ALPM] upgraded libreoffice-fresh (26.2.5-1 -> 26.2.5-2)
[2026-08-21T07:23:23+0700] [ALPM] upgraded libwbclient (2:4.24.5-1 -> 2:4.24.6-1)
[2026-08-21T07:23:23+0700] [ALPM] upgraded libyuv (r2426+464c51a03-1 -> r2921+644251f25-1)
[2026-08-21T07:23:23+0700] [ALPM] upgraded linux-firmware-whence (20260622-1 -> 20260810-2)
[2026-08-21T07:23:23+0700] [ALPM] upgraded linux-firmware-amdgpu (20260622-1 -> 20260810-2)
[2026-08-21T07:23:23+0700] [ALPM] upgraded linux-firmware-atheros (20260622-1 -> 20260810-2)
[2026-08-21T07:23:23+0700] [ALPM] upgraded linux-firmware-broadcom (20260622-1 -> 20260810-2)
[2026-08-21T07:23:23+0700] [ALPM] upgraded linux-firmware-cirrus (20260622-1 -> 20260810-2)
[2026-08-21T07:23:23+0700] [ALPM] upgraded linux-firmware-intel (20260622-1 -> 20260810-2)
[2026-08-21T07:23:23+0700] [ALPM] upgraded linux-firmware-mediatek (20260622-1 -> 20260810-2)
[2026-08-21T07:23:24+0700] [ALPM] upgraded linux-firmware-nvidia (20260622-1 -> 20260810-2)
[2026-08-21T07:23:24+0700] [ALPM] upgraded linux-firmware-other (20260622-1 -> 20260810-2)
[2026-08-21T07:23:24+0700] [ALPM] upgraded linux-firmware-radeon (20260622-1 -> 20260810-2)
[2026-08-21T07:23:24+0700] [ALPM] upgraded linux-firmware-realtek (20260622-1 -> 20260810-2)
[2026-08-21T07:23:24+0700] [ALPM] upgraded linux-firmware (20260622-1 -> 20260810-2)
[2026-08-21T07:23:24+0700] [ALPM] upgraded mkinitcpio (41-4 -> 41.1-1)
[2026-08-21T07:23:25+0700] [ALPM] upgraded linux-lts (6.18.43-3 -> 6.18.45-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded linux-lts-headers (6.18.43-3 -> 6.18.45-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded modemmanager-qt (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded nano (9.1-1 -> 9.2-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded wpa_supplicant (2:2.11-5 -> 2:2.12-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded procps-ng (4.0.6-3 -> 4.0.7-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded networkmanager-qt (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded nmap (7.99-3 -> 7.991-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded nvidia-open (610.57.04-5 -> 610.57.04-6)
[2026-08-21T07:23:27+0700] [ALPM] upgraded nvidia-open-lts (1:610.57.04-4 -> 1:610.57.04-6)
[2026-08-21T07:23:27+0700] [ALPM] upgraded qqc2-desktop-style (6.28.0-1 -> 6.29.0-2)
[2026-08-21T07:23:27+0700] [ALPM] upgraded prison (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded qt6-location (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded qt6-virtualkeyboard (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:27+0700] [ALPM] upgraded smbclient (2:4.24.5-1 -> 2:4.24.6-1)
[2026-08-21T07:23:28+0700] [ALPM] upgraded plasma-workspace (6.7.4-1 -> 6.7.4-3)
[2026-08-21T07:23:28+0700] [ALPM] upgraded plasma-integration (6.7.4-1 -> 6.7.4-3)
[2026-08-21T07:23:28+0700] [ALPM] upgraded qt6-websockets (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:28+0700] [ALPM] upgraded powerdevil (6.7.4-1 -> 6.7.4-3)
[2026-08-21T07:23:28+0700] [ALPM] upgraded qt6-webchannel (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:28+0700] [ALPM] upgraded qt6-webengine (6.11.1-5 -> 6.11.2-1)
[2026-08-21T07:23:28+0700] [ALPM] upgraded purpose (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:28+0700] [ALPM] upgraded python-shtab (1.10.0-1 -> 1.11.0-1)
[2026-08-21T07:23:28+0700] [ALPM] upgraded qt6-imageformats (6.11.1-1 -> 6.11.2-1)
[2026-08-21T07:23:29+0700] [ALPM] upgraded rsync (3.4.4-1 -> 3.5.0-1)
[2026-08-21T07:23:29+0700] [ALPM] upgraded simdjson (1:4.6.6-1 -> 1:4.6.7-1)
[2026-08-21T07:23:29+0700] [ALPM] upgraded source-highlight (3.1.9-18 -> 3.1.9-19)
[2026-08-21T07:23:29+0700] [ALPM] upgraded threadweaver (6.28.0-1 -> 6.29.0-1)
[2026-08-21T07:23:29+0700] [ALPM] upgraded tmux (3.7_b-1 -> 3.7_c-1)
[2026-08-21T07:23:29+0700] [ALPM] upgraded zip (3.0-13 -> 3.0-14)
[2026-08-21T22:54:01+0700] [ALPM] installed go (2:1.27.0-1)
[2026-08-22T09:36:44+0700] [ALPM] installed ethtool (1:7.1-1)
[2026-08-22T12:15:08+0700] [ALPM] installed tailscale (1.102.3-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded tar (1.35-3 -> 1.35-5)
[2026-08-22T12:15:08+0700] [ALPM] upgraded ark (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded baloo-widgets (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded libkexiv2 (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded kio-extras (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded dolphin (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded ffmpegthumbs (26.04.3-3 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded libkdcraw (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded signon-kwallet-extension (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded kaccounts-integration (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded gwenview (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded kate (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded konsole (26.04.3-1 -> 26.08.0-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded libnm (1.58.0-1 -> 1.58.1-1)
[2026-08-22T12:15:08+0700] [ALPM] upgraded lib32-libnm (1.58.0-1 -> 1.58.1-1)
[2026-08-22T12:15:09+0700] [ALPM] upgraded linux-lts (6.18.45-1 -> 6.18.45-2)
[2026-08-22T12:15:12+0700] [ALPM] upgraded linux-lts-headers (6.18.45-1 -> 6.18.45-2)
[2026-08-22T12:15:12+0700] [ALPM] upgraded networkmanager (1.58.0-1 -> 1.58.1-1)
[2026-08-22T12:15:12+0700] [ALPM] upgraded okular (26.04.3-1 -> 26.08.0-1)
[2026-08-24T07:28:01+0700] [ALPM] removed jdk21-openjdk (21.0.12.1.u1-1)
[2026-08-24T12:13:49+0700] [ALPM] installed github-cli (2.98.0-1)
```

### Current Disk Layout

```txt
NAME          SIZE FSTYPE LABEL       MOUNTPOINTS
zram0         7.5G swap   zram0       [SWAP]
nvme0n1     476.9G                    
├─nvme0n1p1   200M vfat               /efi
├─nvme0n1p2    16M                    
├─nvme0n1p3  99.2G ntfs               
├─nvme0n1p4   846M ntfs               
├─nvme0n1p5     1G vfat   ARCHBOOT    /boot
└─nvme0n1p6 375.7G btrfs  ARCHROOT    /var/log
                                      /var/cache
                                      /home
                                      /
nvme1n1     476.9G                    
└─nvme1n1p1 476.9G ntfs   Data Shared /mnt/data-shared
```

### Current Kernel

```txt
Linux archlinux 6.18.45-2-lts #1 SMP PREEMPT_DYNAMIC Fri, 21 Aug 2026 22:45:28 +0000 x86_64 GNU/Linux
```

<!-- AUTO_PACKAGE_REPORT_END -->
