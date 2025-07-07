# Secure_Arch_Install_Oculink_eGPU
Installation Steps for a new Lenovo Thinkbook TGX (Oculink) Security Enhanhed Arch Gnome Wayland Focus

# Arch Linux Setup Action Plan for Lenovo ThinkBook 14+ 2025

This action plan outlines the steps to install and configure **Arch Linux** on a **Lenovo ThinkBook 14+ 2025 Intel Core Ultra 7 255H without dGPU**, **GNOME Wayland**, **BTRFS**, **LUKS2**, **systemd-boot with UKI**, **Secure Boot**, and an **OCuP4V2 OCuLink GPU Dock ReDriver (Nvidia 5070 TI)**, including privacy and security hardening measures. This laptop has two M.2, we will have Windows in a slot to help updating BIOS and Firmware.

## Step 1: Verify Hardware
   - Access UEFI BIOS (F2 at boot):
     - Enable TPM 2.0, Secure Boot, Resizable BAR, SVM/VT-x, and Intel VT-d (IOMMU).
     - Check for “Hybrid Graphics” or “PCIe Hotplug” options.
     - Set a strong UEFI BIOS password, store in Bitwarden, and disable legacy boot.
     - Check for PCIe hotplug settings.
    
## Step 2: Install Windows on Primary NVMe M.2 (/dev/nvme0n1)
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

## Step 3: Prepare Installation Media
  - Download the latest Arch Linux ISO from [archlinux.org](https://archlinux.org/download/).
  - Verify the ISO and create a bootable USB (e.g., using `dd` or Ventoy).

## Step 4: Pre-Arch Installation Steps
**Boot Arch Live USB (disable Secure Boot temporarily in UEFI)**

  **a) Optimize Mirrorlist**:
  - Install `reflector`: `pacman -S reflector pacman-contrib`.
  - Update mirrorlist: `reflector --latest 10 --sort rate --save /etc/pacman.d/mirrorlist`.
  - Enable `reflector.timer`: `systemctl enable --now reflector.timer`.
  
  **b) Partition the Second NVMe M.2 (/dev/nvme1n1)**:
  - Use `fdisk /dev/nvme1n1` to create:
    - Partition 1: 1 GB ESP (`/boot`), type EFI System Partition, formatted as FAT32: `mkfs.fat -F32 /dev/nvme1n1p1`.
    - Partition 2: Remaining space for BTRFS (LUKS2-encrypted).
      
  **c) Set Up LUKS2 Encryption**:
  - Encrypt the BTRFS partition:
    - cryptsetup luksFormat --type luks2 --tpm2-device=auto --tpm2-pcrs=0+1+2+3+4+5+6+7 /dev/nvme1n1p2
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
    - mount -o subvol=@,compress=zstd:3,ssd,autodefrag /dev/mapper/cryptroot /mnt
    - mkdir -p /mnt/{boot,windows-efi,snapshots,home,data,var,var/lib,var/log,swap,srv}
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
  - Generate `fstab`: `genfstab -U /mnt >> /mnt/etc/fstab`.
  
  **e) Configure Swap File**:
  - Create a swap file on `@swap` subvolume:
    - mount -o subvol=@swap,nodatacow,compress=no /dev/mapper/cryptroot /mnt/swap
    - mkdir -p /mnt/swap
    - touch /mnt/swap/swapfile
    - chattr +C /mnt/swap/swapfile
    - fallocate -l 16G /mnt/swap/swapfile
    - chmod 600 /mnt/swap/swapfile
    - mkswap /mnt/swap/swapfile
    - swapon /mnt/swap/swapfile
  - Get swap file offset for hibernation:
    - SWAP_OFFSET=$(filefrag -v /mnt/swap/swapfile | grep "logical_offset: 0" | awk '{print $4}')
  - Add to `fstab`:
    - echo "/swap/swapfile none swap defaults 0 0" >> /mnt/etc/fstab
  - Enable `fstrim` for SSD maintenance:
    - systemctl enable --now fstrim.timer

## Step 5: **Install Arch Linux in the (/dev/nvme1n1) (re-enable Secure Boot in UEFI)**
  - Install base system: pacstrap /mnt base linux linux-firmware intel-ucode (linux-hardened will not be adopted because it can cause problems with the Nvidia eGPU)
  - Second part of the base system: genfstab -U /mnt >> /mnt/etc/fstab
  - Edit /mnt/etc/fstab to secure mounts:
    - /dev/mapper/cryptroot / btrfs subvol=@,compress=zstd 0 0
    - /dev/mapper/cryptroot /home btrfs subvol=@home,compress=zstd,nosuid,nodev 0 0
    - /dev/mapper/cryptroot /data btrfs subvol=@data,compress=zstd,nosuid,nodev 0 0
    - /dev/nvme0n1p1 /boot vfat defaults 0 2
    - tmpfs /tmp tmpfs defaults,nosuid,nodev 0 0
    - tmpfs /var/tmp tmpfs defaults,nosuid,nodev 0 0
    - tmpfs /run/shm tmpfs defaults,nosuid 0 0
  - Chroot into the system: arch-chroot /mnt

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
     
## Step 8: **Configure systemd-boot with UKI**
  **a) Install systemd-boot:**
  - Install to Linux ESP (`/dev/nvme1n1p1`):
    - pacman -S systemd-boot
    - bootctl install
       
  **b) Configure mkinitcpio for UKI:**
  - Edit `/etc/mkinitcpio.conf`:
    - BINARIES=(/usr/lib/systemd/systemd-cryptsetup /usr/bin/btrfs)
    - HOOKS=(base systemd autodetect modconf block plymouth sd-encrypt resume filesystems)
    - MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm nvme pciehp)
  - Generate UKI:
    - mkinitcpio -P
   
  **c) Create Boot Entries:**
  - Get UUIDs:
    - LUKS_UUID=$(cryptsetup luksUUID /dev/nvme1n1p2)
  - Create Arch boot entry:
    - echo -e "title Arch Linux\nlinux /EFI/Linux/arch.efi\noptions rd.luks.name=$LUKS_UUID=cryptroot root=/dev/mapper/cryptroot rw quiet resume_offset=$SWAP_OFFSET nvidia-drm.modeset=1 rcutree.rcu_idle_gp_delay=1" > /boot/loader/entries/arch.conf
  - Create Windows boot entry:
    - echo -e "title Windows\nloader /EFI/Microsoft/Boot/bootmgfw.efi" > /boot/loader/entries/windows.conf
  
  **d) Set Boot Order:**
   - Pin Arch first:
     - efibootmgr --create --disk /dev/nvme1n1 --part 1 --loader /EFI/Linux/arch.efi --label "Arch Linux" --unicode
     - efibootmgr --bootorder $(efibootmgr | grep "Arch Linux" | cut -d' ' -f1 | sed 's/Boot//'),$(efibootmgr | grep "Windows" | cut -d' ' -f1 | sed 's/Boot//')
  
   **e) Create Fallback Bootloader:**
   - Create minimal UKI config: `/etc/mkinitcpio-minimal.conf` (copy `/etc/mkinitcpio.conf`, remove non-essential hooks).
   - Generate minimal UKI:
     - mkinitcpio -P -c /etc/mkinitcpio-minimal.conf
   - Create GRUB USB for recovery:
     - pacman -S grub
     - grub-install --target=x86_64-efi --efi-directory=/mnt/usb
  
## Step 9: **Set Up TPM and LUKS2**

  **a) Install tpm2-tools:**
   - pacman -S tpm2-tools

  **b) Bind LUKS2 Key to TPM:**
  - Enroll LUKS key:
   - systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+1+2+3+4+5+6+7 /dev/nvme1n1p2
   - cryptsetup luksAddKey /dev/nvme1n1p2
   - dd bs=512 count=4 if=/dev/random of=/root/luks-keyfile iflag=fullblock
   - cryptsetup luksAddKey /dev/nvme1n1p2 /root/luks-keyfile
   - chmod 600 /root/luks-keyfile
  - Back up keyfile to a secure USB.

  **c) Update crypttab:**
   - echo "cryptroot /dev/nvme1n1p2 none luks,tpm2-device=auto,tpm2-pcrs=0+1+2+3+4+5+6+7" >> /etc/crypttab

  **c) Enable Plymouth:**
   - Install and configure:
    - pacman -S plymouth
    - plymouth-set-default-theme -R bgrt
   - Ensure `plymouth` is before `sd-encrypt` in `/etc/mkinitcpio.conf` HOOKS and regenerate:
    - mkinitcpio -P  

## Step 10: **Configure Secure Boot** 

  **a) Install sbctl:**
   - pacman -S sbctl

  **b) Generate and Enroll MOKs:**
   - sbctl create-keys
   - sbctl enroll-keys
   - sbctl sign -s /boot/loader/entries/arch.conf
  - Reboot and enroll MOK in UEFI firmware.

  **c) Sign UKI and Nvidia Modules:**
   - sbctl sign /boot/EFI/Linux/arch.efi
   - pacman -S nvidia-dkms nvidia-utils nvidia-settings nvidia-prime
   - sbctl sign /usr/lib/modules/$(uname -r)/updates/dkms/nvidia.ko
   - sbctl sign /usr/lib/modules/$(uname -r)/updates/dkms/nvidia-modeset.ko
   - sbctl sign /usr/lib/modules/$(uname -r)/updates/dkms/nvidia-drm.ko
   - sbctl sign /usr/lib/modules/$(uname -r)/updates/dkms/nvidia-uvm.ko

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
     - Exec = /usr/bin/bash -c "[ -x /usr/bin/sbctl ] && { sbctl sign /usr/lib/modules/$(uname -r)/updates/dkms/nvidia/*.ko; sbctl sign /boot/EFI/Linux/arch.efi; sbctl sign /usr/lib/systemd/boot/efi/systemd-bootx64.efi; }"
     - EOF   
     
## Step 11: **Install and Configure DE and Applications**

  **a) Install yay and Gnome:**
   - pacman -S yay Gnome
  - Reboot and Start Gnome

  **b) Install the applications: (Review installations steps in each application instructions websites before installing it)**
  - Install `thinklmi` to verify BIOS settings:
   - yay -S thinklmi
  - Check BIOS settings: sudo thinklmi
  - Install core applications:
   - yay -S gdm nautilus helix zsh zellij yazi tlp powertop cpupower upower ufw apparmor flatpak flatseal bitwarden blender krita gimp gcc gdb rustup python-pygobject git fwupd lynis usbguard sshguard rkhunter ripgrep fd eza gstreamer gst-plugins-good gst-plugins-bad gst-plugins-ugly ffmpeg gst-libav fprintd auditd chkrootkit libva-vdpau-driver libva-nvidia-driver zram-generator xdg-ninja mullvad-browser windscribe-vpn bubblejail
  - Install applications via Flatpak:
   - flatpak install flathub lollypop steam element-desktop Brave Tor Standard-Notes

 **c) Install Fonts**
  - yay -S ttf-inter ttf-roboto noto-fonts ttf-ubuntu-font-family ttf-ibm-plex ttf-ubuntu-mono-nerd ttf-jetbrains-mono ttf-fira-code ttf-cascadia-code ttf-hack ttf-iosevka ttf-source-code-pro ttf-dejavu ttf-anonymous-pro catppuccin-cursors-mocha nerd-fonts-jetbrains-mono
  
