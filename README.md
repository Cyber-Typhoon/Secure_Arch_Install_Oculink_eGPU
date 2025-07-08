# Secure_Arch_Install_Oculink_eGPU
Installation Steps for a new Lenovo Thinkbook TGX (Oculink) Security Enhanhed Arch Gnome Wayland Focus

# Arch Linux Setup Action Plan for Lenovo ThinkBook 14+ 2025

This action plan outlines the steps to install and configure **Arch Linux** on a **Lenovo ThinkBook 14+ 2025 Intel Core Ultra 7 255H without dGPU**, **GNOME Wayland**, **BTRFS**, **LUKS2**, **TPM2**, **systemd-boot with UKI**, **Secure Boot**, and an **OCuP4V2 OCuLink GPU Dock ReDriver (Nvidia 5070 TI - testing first with AMD Vega 56)**, including privacy and security hardening measures. This laptop has two M.2, we will have Windows in a slot to help updating BIOS and Firmware.

Observation: Not adopting linux-hardened kernel because of complexity in the setup.

## Step 1: **Verify Hardware**
   - Access UEFI BIOS (F2 at boot):
     - Enable TPM 2.0, Secure Boot, Resizable BAR, SVM/VT-x, and Intel VT-d (IOMMU).
     - Check for “Hybrid Graphics” or “PCIe Hotplug” options.
     - Set a strong UEFI BIOS password, store in Bitwarden, and disable legacy boot.
    
## Step 2: **Install Windows on Primary NVMe M.2 (/dev/nvme0n1)**
   - Install Windows 11 Pro on the primary NVMe M.2.
   - Follow some of the installations Privacy advises from the Privacy Guides Wiki [Minimizing Windows 11 Data Collection](https://discuss.privacyguides.net/t/minimizing-windows-11-data-collection/28193)
   - Allow Windows to create its default partitions, including a ~100-300 MB EFI System Partition (ESP) at /dev/nvme0n1p1.
   - Review the guides for additional Privacy on the post installation [Group Police](https://www.privacyguides.org/en/os/windows/group-policies/), [Windows Privacy Settings](https://discuss.privacyguides.net/t/windows-privacy-settings/27333) and [Windows Post-Install Hardening Guide](https://discuss.privacyguides.net/t/windows-post-install-hardening-guide/27335)
   - Update BIOS and Firmware (TPM updates) via Lenovo Vantage Application or Website
   - Disable Windows Fast Startup to prevent ESP lockout (Powershell): powercfg /h off
   - Verify TPM 2.0 is active: `tpm.msc`. Clear TPM if previously provisioned (Powershell): tpm.msc
   - Verify Windows boots correctly and **check Resizable BAR sizes in Device Manager** or `lspci -vv | grep -i region` (in Linux later).
   - Verify NVMe drives: `lsblk` in a live environment or **Windows Disk Management**.
   - Test `fwupd` in Windows (if supported) for Linux compatibility: `fwupdmgr refresh`.
   - **optional** Shrink Windows partition using Disk Management to ensure space for Linux ESP if using a single ESP (**optional; plan uses separate ESPs**).

## Step 3: **Prepare Installation Media**
  - Download the latest Arch Linux ISO from [archlinux.org](https://archlinux.org/download/).
  - Verify the ISO and create a bootable USB (e.g., using `dd` or Ventoy).

## Step 4: **Pre-Arch Installation Steps**
**Boot Arch Live USB (disable Secure Boot temporarily in UEFI)**

  **a) Partition the Second NVMe M.2 (/dev/nvme1n1 - remove single quotes)**:
  - cat /sys/firmware/efi/fw_platform_size  # Should return 64
  - parted /dev/nvme1n1
  - mklabel gpt
  - mkpart ESP fat32 1MiB 1GiB
  - set 1 esp on
  - mkpart primary 1GiB 100%
  - quit
  - lsblk -f /dev/nvme0n1 /dev/nvme1n1  # Confirm /dev/nvme0n1p1 (Windows ESP) and /dev/nvme1n1p1 (Arch ESP)
  - efibootmgr  # Check if UEFI recognizes both ESPs
    
  **b) Format ESP**:
  - mkfs.fat -F32 /dev/nvme1n1p1:
      
  **c) Set Up LUKS2 Encryption**:
  - Encrypt the BTRFS partition:
    - cryptsetup luksFormat --type luks2 /dev/nvme1n1p2
    - cryptsetup luksOpen /dev/nvme1n1p2 cryptroot
  
  **d) Create BTRFS Filesystem and Subvolumes**:
  - Format as BTRFS:
    - mkfs.btrfs /dev/mapper/cryptroot
    - mount /dev/mapper/cryptroot /mnt
  - Create subvolumes for `@`, `@snapshots`, `@home`, `@data`, `@var`, `@var_lib`, `@log`, `@swap`, `@srv`:
    - btrfs subvolume create /mnt/@
    - btrfs subvolume create /mnt/@snapshots
    - btrfs subvolume create /mnt/@home
    - btrfs subvolume create /mnt/@data
    - btrfs subvolume create /mnt/@var
    - btrfs subvolume create /mnt/@var_lib
    - btrfs subvolume create /mnt/@log
    - btrfs subvolume create /mnt/@swap
    - btrfs subvolume create /mnt/@srv
    - umount /mnt
  - Mount subvolumes with optimized options:
    - mkdir -p /mnt/{boot,windows-efi,snapshots,home,data,var,var/lib,var/log,swap,srv}
    - mount -o subvol=@,compress=zstd:3,ssd,autodefrag /dev/mapper/cryptroot /mnt
    - mount -o subvol=@snapshots,ssd /dev/mapper/cryptroot /mnt/snapshots
    - mount -o subvol=@home,compress=zstd:3,ssd,autodefrag /dev/mapper/cryptroot /mnt/home
    - mount -o subvol=@data,compress=zstd:3,ssd,autodefrag /dev/mapper/cryptroot /mnt/data
    - mount -o subvol=@var,nodatacow,compress=no /dev/mapper/cryptroot /mnt/var
    - mount -o subvol=@var_lib,nodatacow,compress=no /dev/mapper/cryptroot /mnt/var/lib
    - mount -o subvol=@log,nodatacow,compress=no /dev/mapper/cryptroot /mnt/var/log
    - mount -o subvol=@swap,nodatacow,compress=no /dev/mapper/cryptroot /mnt/swap
    - mount -o subvol=@srv,compress=zstd:3,ssd /dev/mapper/cryptroot /mnt/srv
    - mount /dev/nvme1n1p1 /mnt/boot
    - mount /dev/nvme0n1p1 /mnt/windows-efi
  - Add tmpfs for `/tmp`:
    - echo "tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0" >> /mnt/etc/fstab
  - Generate fstab:
    - genfstab -U /mnt > /mnt/etc/fstab
    - cat /mnt/etc/fstab  # Check for duplicates or incorrect UUIDs
  
  **e) Configure Swap File**:
  - Create a swap file on `@swap` subvolume:
    - mount -o subvol=@swap,nodatacow,compress=no /dev/mapper/cryptroot /mnt/swap
    - touch /mnt/swap/swapfile
    - chattr +C /mnt/swap/swapfile
    - fallocate -l 24G /mnt/swap/swapfile
    - chmod 600 /mnt/swap/swapfile
    - mkswap /mnt/swap/swapfile
    - swapon /mnt/swap/swapfile
    - SWAP_OFFSET=$(filefrag -v /mnt/swap/swapfile | awk '{if($1=="0:"){print $4}}' | sed 's/\.\.//')
    - echo $SWAP_OFFSET > /mnt/swap_offset
    - swapon -d /mnt/swap/swapfile
    - swapon --show  # Verify swap is active
  - Enable `fstrim` for SSD maintenance:
    - systemctl enable --now fstrim.timer
  - Mirrorlist Before pacstrap
    - pacman -Sy reflector  
    - reflector --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
  - Copy DNS into the new system so it can resolve mirrors
    - cp /etc/resolv.conf /mnt/etc/resolv.conf  
  - Before writing the fstab, **get the UUID of your Arch ESP partition (/dev/nvme1n1p1)**
    - ARCH_ESP_UUID=$(blkid -s UUID -o value /dev/nvme1n1p1)
   
**f) Check network**:
  - ping -c 3 archlinux.org
  - nmcli device wifi connect <SSID> password <password>  # If using Wi-Fi

## Step 5: **Install Arch Linux in the (/dev/nvme1n1) (re-enable Secure Boot in UEFI)**
  - Install base system:
    - pacstrap /mnt base linux linux-firmware intel-ucode zsh 
  - Edit /mnt/etc/fstab to secure mounts:
  - cat << 'EOF' > /mnt/etc/fstab
    - /dev/mapper/cryptroot /btrfs subvol=@,compress=zstd:3,ssd,autodefrag,noatime 0 0
    - /dev/mapper/cryptroot /home btrfs subvol=@home,compress=zstd:3,ssd,autodefrag,noatime,nosuid,nodev 0 0
    - /dev/mapper/cryptroot /data btrfs subvol=@data,compress=zstd:3,ssd,autodefrag,noatime,nosuid,nodev 0 0
    - /dev/mapper/cryptroot /.snapshots btrfs subvol=@snapshots,ssd,noatime 0 0
    - /dev/mapper/cryptroot /var btrfs subvol=@var,nodatacow,compress=no,noatime 0 0
    - /dev/mapper/cryptroot /var/lib btrfs subvol=@var_lib,nodatacow,compress=no,noatime 0 0
    - /dev/mapper/cryptroot /var/log btrfs subvol=@log,nodatacow,compress=no,noatime 0 0
    - /dev/mapper/cryptroot /srv btrfs subvol=@srv,compress=zstd:3,ssd,noatime 0 0
    - /dev/mapper/cryptroot /swap btrfs subvol=@swap,nodatacow,compress=no,noatime 0 0
    - /swap/swapfile none swap defaults 0 0
    - UUID=${ARCH_ESP_UUID} /boot vfat umask=0077 0 2 **use the UUID of your Arch ESP partition**
    - tmpfs /tmp tmpfs defaults,noatime,nosuid,nodev,mode=1777 0 0
    - tmpfs /var/tmp tmpfs defaults,noatime,nosuid,nodev,mode=1777 0 0
    - tmpfs /run/shm tmpfs defaults,noatime,nosuid 0 0
    - EOF
  - Chroot into the system:
    - arch-chroot /mnt
  - Keyring initialization step
    - pacman-key --init
    - pacman-key --populate archlinux

## Step 6: **Set timezone, locale, and hostname**
  - ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
  - hwclock --systohc
  - echo 'en_US.UTF-8 UTF-8' > /etc/locale.gen
  - locale-gen
  - echo 'LANG=en_US.UTF-8' > /etc/locale.conf
  - echo 'thinkbook' > /etc/hostname
  - echo '127.0.0.1 localhost' >> /etc/hosts
  - echo '::1 localhost' >> /etc/hosts
  - echo '127.0.1.1 thinkbook.localdomain thinkbook' >> /etc/hosts

## Step 7: **Create User Account**
  - Set root password:
    - passwd 
  - Create a user with Zsh as the default shell:
    - useradd -m -G wheel -s /usr/bin/zsh <username>
    - passwd <username>
  - Configure `sudo`:
    - visudo # Uncomment %wheel ALL=(ALL:ALL) ALL

## Step 8: **Set Up TPM and LUKS2**

  **a) Install tpm2-tools:**
   - pacman -S tpm2-tools

  **b) Bind LUKS2 Key to TPM:**
  - Enroll LUKS key:
   - systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/nvme1n1p2
   - dd bs=512 count=4 if=/dev/random of=/root/luks-keyfile iflag=fullblock
   - cryptsetup luksAddKey /dev/nvme1n1p2 /root/luks-keyfile
   - chmod 600 /root/luks-keyfile
   - systemd-cryptenroll --tpm2-device=auto --test /dev/nvme1n1p2
  - Update crypttab:
   - echo "cryptroot /dev/nvme1n1p2 none luks,tpm2-device=auto,tpm2-pcrs=0+7" >> /etc/crypttab 
  - Back up keyfile to a secure USB.

  **c) Enable Plymouth:**
   - Install and configure:
    - pacman -S plymouth
    - plymouth-set-default-theme -R bgrt
   - Ensure `plymouth` is before `sd-encrypt` in `/etc/mkinitcpio.conf` HOOKS and regenerate:
    - mkinitcpio -P  
     
## Step 9: **Configure systemd-boot with UKI**
  **a) Install systemd-boot:**
    - pacman -S systemd-boot
    - bootctl install
       
  **b) Configure mkinitcpio for UKI:**
  - Edit `/etc/mkinitcpio.conf`:
  - cat << 'EOF' > /etc/mkinitcpio.conf
    - BINARIES=(/usr/lib/systemd/systemd-cryptsetup /usr/bin/btrfs)
    - HOOKS=(base systemd autodetect modconf block plymouth sd-encrypt resume filesystems)
    - MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm nvme pciehp)
    - EOF
  - Generate UKI:
    - mkinitcpio -P
   
  **c) Create Boot Entries:**
  - LUKS_UUID=$(cryptsetup luksUUID /dev/nvme1n1p2)
  - cat << EOF > /boot/loader/entries/arch.conf
   - title Arch Linux
   - linux /EFI/Linux/arch.efi
   - options rd.luks.name=$LUKS_UUID=cryptroot root=UUID=$ROOT_UUID resume=UUID=$ROOT_UUID resume_offset=$SWAP_OFFSET rw quiet nvidia-drm.modeset=1 rcutree.rcu_idle_gp_delay=1
   - EOF
  - cat << 'EOF' > /boot/loader/entries/windows.conf
   - title Windows
   - loader /EFI/Microsoft/Boot/bootmgfw.efi
   - EOF
  
  **d) Set Boot Order:**
   - Pin Arch first:
     - efibootmgr --create --disk /dev/nvme1n1 --part 1 --loader /EFI/Linux/arch.efi --label "Arch Linux" --unicode
     - efibootmgr --bootorder $(efibootmgr | grep "Arch Linux" | cut -d' ' -f1 | sed 's/Boot//'),$(efibootmgr | grep "Windows" | cut -d' ' -f1 | sed 's/Boot//')
     - efibootmgr  # Ensure both Arch and Windows entries are listed
  
   **e) Create Fallback Bootloader:**
   - Create minimal UKI config: `/etc/mkinitcpio-minimal.conf` (copy `/etc/mkinitcpio.conf`, remove non-essential hooks).
     - cp /etc/mkinitcpio.conf /etc/mkinitcpio-minimal.conf
     - sed -i 's/HOOKS=(.*)/HOOKS=(base systemd autodetect modconf block sd-encrypt filesystems)/' /etc/mkinitcpio-minimal.conf
   - Generate minimal UKI:
     - mkinitcpio -P -c /etc/mkinitcpio-minimal.conf
   - Create GRUB USB for recovery:
     - **Replace /dev/sdX1 with your USB partition**
     - mkfs.fat -F32 /dev/sdX1
     - mkdir -p /mnt/usb
     - mount /dev/sdX1 /mnt/usb
     - pacman -S grub
     - grub-install --target=x86_64-efi --efi-directory=/mnt/usb
  
## Step 10: **Configure Secure Boot** 

  **a) Install sbctl:**
   - pacman -S sbctl

  **b) Generate and Enroll MOKs:**
   - sbctl create-keys
   - sbctl enroll-keys --tpm-eventlog
   - sbctl sign -s /boot/EFI/Linux/arch.efi
   - sbctl sign -s /usr/lib/systemd/boot/efi/systemd-bootx64.efi
   - **Reboot and enroll keys in UEFI firmware when prompted**

  **c) Sign UKI and Nvidia Modules:**
   - pacman -S nvidia-dkms nvidia-utils nvidia-settings nvidia-prime
   - KERNEL_VERSION=$(ls /usr/lib/modules | grep -E '^[0-9]+\.[0-9]+\.[0-9]+')
   - sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia.ko
   - sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia-modeset.ko
   - sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia-drm.ko
   - sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia-uvm.ko

  **d) Automate Signing:**
   - mkdir -p /etc/pacman.d/hooks
   - cat << 'EOF' > /etc/pacman.d/hooks/99-secure-boot-signing.hook
     - [Trigger]
     - Operation = Install
     - Type = Package
     - Target = nvidia-dkms
     - Target = linux
     - Target = systemd
     - [Action]
     - Description = Signing kernel modules and UKI for Secure Boot...
     - When = PostTransaction
     - Exec = /usr/bin/bash -c "[ -x /usr/bin/sbctl ] && { KERNEL_VERSION=$(ls /usr/lib/modules | grep -E '^[0-9]+\.[0-9]+\.[0-9]+'); sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia/*.ko; sbctl sign /boot/EFI/Linux/arch.efi; sbctl sign /usr/lib/systemd/boot/efi/systemd-bootx64.efi; }"
     - EOF

  **- Reboot and enroll MOK in UEFI firmware.**
     
## Step 11: **Install and Configure DE and Applications**

  **a) Install yay and Gnome:**
   - pacman -S yay gnome
   - Install [Alacritty](https://github.com/alacritty/alacritty/blob/master/INSTALL.md) 
  - Reboot and Start gnome

  **b) Install the applications: (Review installations steps in each application instructions websites before installing it)**
  - Install `thinklmi` to verify BIOS settings:
   - yay -S thinklmi
  - Check BIOS settings: sudo thinklmi
  - Install core applications:
   - yay -S gnome-tweaks networkmanager bluez bluez-utils ufw apparmor tlp powertop cpupower upower systemd-timesyncd zsh snapper fapolicyd sshguard rkhunter lynis usbguard aide pacman-notifier mullvad-browser brave-browser tor-browser bitwarden helix zellij yazi blender krita gimp gcc gdb rustup python-pygobject git fwupd xdg-ninja libva-vdpau-driver libva-nvidia-driver zram-generator ripgrep fd eza gstreamer gst-plugins-good gst-plugins-bad gst-plugins-ugly ffmpeg gst-libav fprintd dnscrypt-proxy systeroid rage zoxide jaq atuin gitui glow delta tokei dua tealdeer fzf procs gping dog httpie bottom bandwhich gnome-bluetooth openSnitch
  - Install applications via Flatpak:
   - flatpak install flathub lollypop steam element-desktop Standard-Notes

 **c) Install Fonts**
  - yay -S ttf-inter ttf-roboto noto-fonts ttf-ubuntu-font-family ttf-ibm-plex ttf-ubuntu-mono-nerd ttf-jetbrains-mono ttf-fira-code ttf-cascadia-code ttf-hack ttf-iosevka ttf-source-code-pro ttf-dejavu ttf-anonymous-pro catppuccin-cursors-mocha nerd-fonts-jetbrains-mono

 **d) Enable services:**
  - systemctl enable gdm bluetooth ufw auditd apparmor systemd-timesyncd tlp power-profiles-daemon NetworkManager
  - After enabling all systemd services, run systemctl --failed to check for misconfigurations or missing dependencies.

 **e) Configure Flatseal for Flatpak apps:**
  - flatpak override --user --nofilesystem=host
  - flatpak override --user com.valvesoftware.Steam --device=dri  # Allow eGPU access

 **f) Configure Bubblejail for Alacritty and Conky:**
  - yay -S bubblejail
  - bubblejail create --profile generic-gui-app Alacritty
  - bubblejail create --profile generic-gui-app Conky
  - bubblejail config Alacritty --add-service dri  # Allow eGPU access
  - bubblejail config Alacritty --add-service wayland
  - bubblejail config Conky --add-service dri
  - bubblejail config Conky --add-service wayland

## Step 12: **Configure Power Management, Security and Privacy**

**a) Configure Power Management:**
   - systemctl enable --now tlp
   - systemctl mask power-profiles-daemon
   - echo -e "CPU_SCALING_GOVERNOR=schedutil\nSOUND_POWER_SAVE_ON_AC=0" >> /etc/tlp.conf
     - cat << 'EOF' > /etc/systemd/system/powertop.service
       - [Unit]
       - Description=Powertop Auto-Tune
       - After=multi-user.target
       - [Service]
       - Type=oneshot
       - ExecStart=/usr/bin/powertop --auto-tune
       - RemainAfterExit=yes
       - [Install]
       - WantedBy=multi-user.target
       - EOF
     - systemctl enable --now powertop.service

 **b) Configure Wayland envars:**
   - cat << 'EOF' > /etc/environment
     - MOZ_ENABLE_WAYLAND=1
     - GDK_BACKEND=wayland
     - CLUTTER_BACKEND=wayland
     - QT_QPA_PLATFORM=wayland
     - SDL_VIDEODRIVER=wayland
     - GBM_BACKEND=nvidia-drm
     - LIBVA_DRIVER_NAME=nvidia
     - XWAYLAND_EXTENSION_GLES=1
     - EOF
   - gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"
   - gsettings set org.gnome.desktop.interface scaling-factor 1.25
   - echo 'prime-run %command%' > ~/.config/wayland-nvidia-run

 **c) Configure MAC randomization:**
   - mkdir -p /etc/NetworkManager/conf.d
   - cat << 'EOF' > /etc/NetworkManager/conf.d/00-macrandomize.conf
     - [device]
     - wifi.scan-rand-mac-address=yes
     - [connection]
     - wifi.cloned-mac-address=random
     - EOF
   - systemctl restart NetworkManager
   
 **d) Configure firewall:**
   - systemctl enable --now ufw
   - ufw enable
   - ufw default deny incoming
   - ufw default allow outgoing

 **e) Configure GNOME privacy:**
   - gsettings set org.gnome.desktop.privacy send-software-usage-info false
   - gsettings set org.gnome.desktop.privacy report-technical-problems false

 **f) Configure IP spoofing protection::**
   - cat << 'EOF' > /etc/host.conf
     - order bind,hosts
     - nospoof on
     - EOF
    
 **g) Configure security limits:**
   - cat << 'EOF' >> /etc/security/limits.conf
     - * hard nproc 1000
     - EOF

 **h) Configure auditd:**
   - systemctl enable --now auditd
   - cat << 'EOF' > /etc/audit/rules.d/audit.rules
     - -w /etc/passwd -p wa -k passwd_changes
     - -w /etc/shadow -p wa -k shadow_changes
     - -a always,exit -F arch=b64 -S execve -k exec
     - -e 2
     - EOF
  - systemctl restart auditd

 **i) Configure AppArmor with Nvidia Exceptions:**
   - apparmor_parser -r /etc/apparmor.d/*
   - aa-complain /usr/bin/nvidia-smi /usr/bin/nvidia-settings

 **j) Configure dnscrypt-proxy:**
   - systemctl enable --now dnscrypt-proxy
   - nmcli connection modify <connection_name> ipv4.dns "127.0.0.1" ipv4.ignore-auto-dns yes
   - nmcli connection modify <connection_name> ipv6.dns "::1" ipv6.ignore-auto-dns yes

  **k) Configure `usbguard` with GSConnect exception:**
   - usbguard generate-policy > /etc/usbguard/rules.conf
   - usbguard list-devices # Identify GSConnect device ID
   - usbguard allow-device <device-id>
   - systemctl enable --now usbguard

  **l) Enable `fapolicyd` with Nvidia exceptions, `sshguard`, `rkhunter`:**
   - pacman -S fapolicyd
   - systemctl enable --now fapolicyd
   - echo "/usr/bin/nvidia-smi trusted" >> /etc/fapolicyd/rules.d/90-custom.rules
   - echo "/usr/bin/nvidia-settings trusted" >> /etc/fapolicyd/rules.d/90-custom.rules
   - systemctl restart fapolicyd
   - systemctl enable --now sshguard
   - rkhunter --propupd

  **m) Run Lynis audit and create timer:**
   - lynis audit system
   - cat << 'EOF' > /etc/systemd/system/lynis-audit.timer
     - [Unit]
     - Description=Run Lynis audit weekly
     - [Timer]
     - OnCalendar=weekly
     - Persistent=true
     - [Install]
     - WantedBy=timers.target
     - EOF
   - cat << 'EOF' > /etc/systemd/system/lynis-audit.service
     - [Unit]
     - Description=Run Lynis audit
     - [Service]
     - Type=oneshot
     - ExecStart=/usr/bin/lynis audit system
     - EOF
   - systemctl enable --now lynis-audit.timer
  
  **n) Configure Flatpak permissions:**
   - flatpak override --user --nofilesystem=host

  **o) Configure AIDE:**
   - yay -S aide
   - aide --init
   - mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
   - systemctl enable --now aide-check.timer

  **p) Optimize Mirrorlist**:
  - Install `reflector`:
    - yay -S reflector pacman-contrib`
  - Update mirrorlist:
    - reflector --latest 10 --sort rate --save /etc/pacman.d/mirrorlist
  - Enable `reflector.timer`:
    - systemctl enable --now reflector.timer

  **q) Configure run0:**
   - yay -S run0
   
  **r) Configure sysctl hardening:**
   - cat << 'EOF' > /etc/sysctl.d/99-hardening.conf
     - net.ipv4.conf.default.rp_filter=1
     - net.ipv4.conf.all.rp_filter=1
     - net.ipv4.tcp_syncookies=1
     - net.ipv4.ip_forward=0
     - net.ipv4.conf.all.accept_redirects=0
     - net.ipv6.conf.all.accept_redirects=0
     - net.ipv4.conf.default.accept_redirects=0
     - net.ipv6.conf.default.accept_redirects=0
     - net.ipv4.conf.all.send_redirects=0
     - net.ipv4.conf.default.send_redirects=0
     - net.ipv4.conf.all.accept_source_route=0
     - net.ipv6.conf.all.accept_source_route=0
     - net.ipv4.conf.default.accept_source_route=0
     - net.ipv6.conf.default.accept_source_route=0
     - net.ipv4.conf.all.log_martians=1
     - net.ipv4.icmp_ignore_bogus_error_responses=1
     - net.ipv4.icmp_echo_ignore_broadcasts=1
     - kernel.randomize_va_space=2
     - EOF
   - sysctl -p /etc/sysctl.d/99-hardening.conf

  **s) Audit SUID binaries:**
   - find / -perm -4000 -type f -exec ls -l {} \; > /data/suid_audit.txt

  **t) Configure zram:**
   - cat << 'EOF' > /etc/systemd/zram-generator.conf
     - [zram0]
     - zram-size = ram / 2
     - compression-algorithm = zstd
     - EOF
   - systemctl enable --now systemd-zram-setup@zram0.service

## Step 13: **Configure eGPU (Nvidia and AMD)**
  - Install AMD drivers for first testing
    - yay -S mesa libva-mesa-driver
  - Identify eGPU dock PCI ID:
    - lspci -nn | grep -i "bridge.*oculink" 
  - Verify eGPU detection:
    - lspci | grep -i nvidia
    - lspci | grep -i amd
    - dmesg | grep -i amdgpu
    - ls /sys/class/drm/card*
    - prime-run glxinfo | grep NVIDIA
    - nvidia-smi
  - Add environment variables for Nvidia:
    - echo -e "GBM_BACKEND=nvidia-drm\n__GLX_VENDOR_LIBRARY_NAME=nvidia" >> /etc/environment
  - Create udev rules for hotplugging (The udev rule you've written uses loginctl terminate-user $USER, which will forcefully log you out and close all your applications every time you connect or disconnect the eGPU.Recommendation: Modern GNOME on Wayland has improved hot-plugging support. Your first step should be to test hot-plugging without any custom udev rules. It may already work. **Start with no rule, and only add one if you find it's absolutely necessary.**):
    - cat << 'EOF' > /etc/udev/rules.d/99-nvidia-egpu.rules
    - - **Replace <device_id> below with Nvidia RTX 5070Ti device ID from lspci**
    - ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{device}=="0x<device_id>", RUN+="/usr/bin/bash -c \"modprobe nvidia nvidia_modeset nvidia_uvm nvidia_drm; nvidia-smi --auto-boost-default=0; loginctl terminate-user $USER\""
    - ACTION=="remove", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{device}=="0x<device_id>", RUN+="/usr/bin/bash -c \"modprobe -r nvidia_drm nvidia_uvm nvidia_modeset nvidia; loginctl terminate-user $USER\""
    - EOF
    - udevadm control --reload-rules && udevadm trigger
  - Enable PCIe hotplug:
    - echo "pciehp" | sudo tee /etc/modules-load.d/pciehp.conf
   
## Step 14: **Configure Snapper**
  - Install Snapper:
    - yay -S snapper
  - Create global filter:
    - mkdir -p /etc/snapper/filters
    - echo -e "/home/.cache\n/tmp\n/run" > /etc/snapper/filters/global-filter.txt
  - Create configurations:
    - snapper --config root create-config /
    - snapper --config home create-config /home
    - snapper --config data create-config /data
  - Edit `/etc/snapper/configs/root`:
    - cat << 'EOF' > /etc/snapper/configs/root
      - TIMELINE_CREATE="yes"
      - TIMELINE_CLEANUP="yes"
      - TIMELINE_MIN_AGE="1800"
      - TIMELINE_LIMIT_HOURLY="0"
      - TIMELINE_LIMIT_DAILY="7"
      - TIMELINE_LIMIT_WEEKLY="4"
      - TIMELINE_LIMIT_MONTHLY="6"
      - TIMELINE_LIMIT_YEARLY="0"
      - SUBVOLUME="/"
      - ALLOW_GROUPS=""
      - SYNC_ACL="no"
      - FILTER="/etc/snapper/filters/global-filter.txt"
      - EOF
    - Edit `/etc/snapper/configs/home` and `/etc/snapper/configs/data` similarly, updating `SUBVOLUME` to `/home` and `/data`.
    - Enable Snapper:
      - systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
     
 ## Step 15: **Configure Dotfiles**
  - Install chezmoi:
    - yay -S chezmoi
    - chezmoi init --apply
   
 ## Step 16: **Test the Setup**
  - Reboot and confirm `systemd-boot` shows Arch and Windows entries.
  - Test Arch boot with TPM-based LUKS unlocking and passphrase fallback.
  - Test Windows boot.
  - Test eGPU (AMD first, then Nvidia):
   - **AMD GPU**:
     - lspci | grep -i amd
     - dmesg | grep -i amdgpu
     - ls /sys/class/drm/card*
     - DRI_PRIME=1 glxinfo | grep "OpenGL renderer"
   - **NVIDIA GPU**:
     - lspci | grep -i nvidia
     - nvidia-smi
     - prime-run glxinfo | grep "OpenGL renderer"
 - Test hotplugging:
   -  udevadm monitor
   -  echo "0000:xx:00.0" | sudo tee /sys/bus/pci/devices/0000:xx:00.0/remove
   -  echo 1 | sudo tee /sys/bus/pci/rescan
   -  pkill -HUP gnome-shell
 - Test hibernation:
   - systemctl hibernate
   - dmesg | grep -i "hibernate\|swap" # After resuming, check dmesg for errors
   - filefrag -v /swap/swapfile  # Ensure no fragmentation
 - Test fwupd
   - fwupdmgr refresh
   - fwupdmgr update
 - Test Snapshots
   - snapper --config root create --description "Test snapshot"
   - snapper list
 - Test Timers
   - journalctl -u yay-update.timer
   - journalctl -u snapper-timeline.timer
   - journalctl -u fstrim.timer
   - journalctl -u lynis-audit.timer
 - Stress Test
   - yay -S stress-ng memtester fio
   - stress-ng --cpu 4 --io 2 --vm 2 --vm-bytes 1G --timeout 72h
   - memtester 1024 5
   - fio --name=write_test --filename=/tmp/test --size=1G --rw=write
 - Verify Wayland
   - echo $XDG_SESSION_TYPE
 - Verify Security
   - auditctl -l
   - apparmor_status 
 - Test AUR builds with /tmp (no noexec)
   - yay -S package-name
 - Test Nvidia CUDA apps
   - prime-run blender 

 ## Step 17: **Create Recovery Documentation**
  - Document UEFI password, LUKS passphrase, keyfile location, MOK password, and recovery steps in Bitwarden.
  - Create a recovery USB with Arch ISO, minimal UKI, and `systemd-cryptsetup`.
  - Back up LUKS header and SBCTL keys:
    - cryptsetup luksHeaderBackup /dev/nvme1n1p2 --header-backup-file /path/to/luks-header-backup
    - cp -r /etc/sbctl /path/to/backup/sbctl-keys

 ## Step 18: **Backup Strategy**
  - Local Snapshots:
    - Managed by Snapper for `@`, `@home`, `@data`, excluding `/var`, `/var/lib`, `/log`, `/tmp`, `/run`.
  - Offsite Snapshots:
    - To be refined savig the data in local server - check btrbk and restic

 
