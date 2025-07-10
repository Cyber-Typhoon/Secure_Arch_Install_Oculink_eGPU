# Secure_Arch_Install_Oculink_eGPU
Installation Steps for a new Lenovo Thinkbook TGX (Oculink) Security Enhanhed Arch Gnome Wayland Focus

# Arch Linux Setup Action Plan for Lenovo ThinkBook 14+ 2025

This action plan outlines the steps to install and configure **Arch Linux** on a **Lenovo ThinkBook 14+ 2025 Intel Core Ultra 7 255H without dGPU**, **GNOME Wayland**, **BTRFS**, **LUKS2**, **TPM2**, **Appamor**, **systemd-boot with UKI**, **Secure Boot**, and an **OCuP4V2 OCuLink GPU Dock ReDriver (Nvidia 5070 TI - testing first with AMD Vega 56)**, including privacy and security hardening measures. This laptop has two M.2, we will have Windows in a slot to help updating BIOS and Firmware.

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
   - Disable Windows Bitlocker (Powershell): a) manage-bde -status b) Disable-BitLocker -MountPoint "C:" c) powercfg /a
   - Verify TPM 2.0 is active: `tpm.msc`. Clear TPM if previously provisioned (Powershell): tpm.msc
   - Verify Windows boots correctly and **check Resizable BAR sizes in Device Manager** or wmic path Win32_VideoController get CurrentBitsPerPixel,VideoMemoryType or `dmesg | grep -i "BAR.*size"` (in Linux later).
   - Check Oculink support 'dmidecode -s bios-version'
   - Verify NVMe drives: `lsblk` in a live environment or **Windows Disk Management**.
   - **optional** Shrink Windows partition using Disk Management to ensure space for Linux ESP if using a single ESP (**optional; plan uses separate ESPs**).

## Step 3: **Prepare Installation Media**
  - Download the latest Arch Linux ISO from [archlinux.org](https://archlinux.org/download/).
  - Verify the ISO and create a bootable USB (e.g., using `dd` or Ventoy). gpg --verify archlinux-<version>-x86_64.iso.sig

## Step 4: **Pre-Arch Installation Steps**
**Boot Arch Live USB (disable Secure Boot temporarily in UEFI)**

  **a) Partition the Second NVMe M.2 (/dev/nvme1n1)**:
  - parted /dev/nvme1n1 --script mklabel gpt mkpart ESP fat32 1MiB 1GiB set 1 esp on mkpart crypt btrfs 1GiB 100% align-check optimal 1 quit
  - lsblk -f /dev/nvme0n1 /dev/nvme1n1  # Confirm /dev/nvme0n1p1 (Windows ESP) and /dev/nvme1n1p1 (Arch ESP)
  - efibootmgr  # Check if UEFI recognizes both ESPs
    
  **b) Format ESP**:
  - mkfs.fat -F32 -n ARCH_ESP /dev/nvme1n1p1
      
  **c) Set Up LUKS2 Encryption**:
  - Encrypt the BTRFS partition:
    - cryptsetup luksFormat --type luks2 /dev/nvme1n1p2 --pbkdf pbkdf2 --pbkdf-force-iterations 1000000
    - cryptsetup luksOpen /dev/nvme1n1p2 cryptroot
    - mkfs.btrfs /dev/mapper/cryptroot
    - mount /dev/mapper/cryptroot /mnt
    - dd if=/dev/random of=/mnt/crypto_keyfile bs=512 count=4 iflag=fullblock
    - cryptsetup luksAddKey /dev/nvme1n1p2 /mnt/crypto_keyfile
    - chmod 600 /mnt/crypto_keyfile
    - mkdir -p /mnt/usb
    - lsblk
    - mount /dev/sdX1 /mnt/usb # **Replace sdX1 with USB partition confirmed via lsblk previously executed**
    - cp /mnt/crypto_keyfile /mnt/usb/crypto_keyfile
       
  **d) Create BTRFS Filesystem and Subvolumes**:
  - Format as BTRFS:
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
    - mkdir -p /mnt/{boot,windows-efi,.snapshots,home,data,var,var/lib,var/log,swap,srv}
    - mount -o subvol=@,compress=zstd:3,ssd,autodefrag /dev/mapper/cryptroot /mnt
    - mount -o subvol=@snapshots,ssd,noatime /dev/mapper/cryptroot /mnt/.snapshots
    - mount -o subvol=@home,compress=zstd:3,ssd,autodefrag /dev/mapper/cryptroot /mnt/home
    - mount -o subvol=@data,compress=zstd:3,ssd,autodefrag /dev/mapper/cryptroot /mnt/data
    - mount -o subvol=@var,nodatacow,compress=no,noatime /dev/mapper/cryptroot /mnt/var
    - mount -o subvol=@var_lib,nodatacow,compress=no,noatime /dev/mapper/cryptroot /mnt/var/lib
    - mount -o subvol=@log,nodatacow,compress=no,noatime /dev/mapper/cryptroot /mnt/var/log
    - mount -o subvol=@swap,nodatacow,compress=no,noatime /dev/mapper/cryptroot /mnt/swap
    - mount -o subvol=@srv,compress=zstd:3,ssd /dev/mapper/cryptroot /mnt/srv
    - mount /dev/nvme1n1p1 /mnt/boot
    - mount /dev/nvme0n1p1 /mnt/windows-efi
  - Generate fstab and Add tmpfs for `/tmp`:
    - ARCH_ESP_UUID=$(blkid -s UUID -o value /dev/nvme1n1p1)
    - WINDOWS_ESP_UUID=$(blkid -s UUID -o value /dev/nvme0n1p1)
    - ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/cryptroot)
    - mkdir -p /mnt/etc
    - genfstab -U /mnt | tee /mnt/etc/fstab
  - Harden subvol log
    - Edit with Nano `/mnt/etc/fstab` and ensure the `/var/log` entry is similar to: `UUID=$ROOT_UUID /var/log btrfs rw,noatime,nodatacow,noexec,subvol=@log 0 0`
    - cat /mnt/etc/fstab  # Check for duplicates or incorrect UUIDs
  
  **e) Configure Swap File**:
  - Create a swap file on `@swap` subvolume:
    - mount -o subvol=@swap,nodatacow,compress=no /dev/mapper/cryptroot /mnt/swap
    - touch /mnt/swap/swapfile
    - chattr +C /mnt/swap/swapfile
    - fallocate -l 24G /mnt/swap/swapfile || { echo "fallocate failed"; exit 1; }
    - chmod 600 /mnt/swap/swapfile
    - mkswap /mnt/swap/swapfile || { echo "mkswap failed"; exit 1; }
    - SWAP_OFFSET=$(btrfs inspect-internal map-swapfile -r /mnt/swap/swapfile)
    - echo $SWAP_OFFSET > /mnt/etc/swap_offset
    - umount /mnt/swap 
    - echo "/swap/swapfile none swap defaults,discard=async,noatime,resume_offset=$SWAP_OFFSET 0 0" >> /mnt/etc/fstab
    - #Manually edit /boot/loader/entries/arch.conf later to include resume_offset=$SWAP_OFFSET
    - cat /mnt/etc/fstab

  - Edit with Nano /mnt/etc/fstab
      - Review existing entries (for `/`, `/boot`, `/home`, etc.) and adjust mount options for BTRFS subvolumes (e.g., `compress=zstd:3`, `ssd`, `nodatacow`, `noatime`).
      - Adjust ESP mount options:
      - For the Arch ESP (`/boot`), find the line added by `genfstab` and ensure `umask=0077` is present. It will look something like: `UUID=<ARCH_ESP_UUID_VALUE> /boot vfat defaults 0 2`. Change `defaults` to `umask=0077`:  
        - UUID=$ARCH_ESP_UUID /boot vfat umask=0077 0 2
      - For the Windows ESP (`/windows-efi`), find the line added by `genfstab` and change its options from `defaults` to `noauto,x-systemd.automount,umask=0077`. It will look something like: `UUID=<WINDOWS_ESP_UUID_VALUE> /windows-efi vfat defaults 0 2`: 
        - UUID=$WINDOWS_ESP_UUID /windows-efi vfat noauto,x-systemd.automount,umask=0077 0 2
      - blkid | grep -E 'nvme0n1p1|nvme1n1p1' #verify each ESP line and UUID
      - Check the other entries:
       - UUID=$ROOT_UUID / btrfs subvol=@,compress=zstd:3,ssd,autodefrag,noatime 0 0
       - UUID=$ROOT_UUID /.snapshots btrfs subvol=@snapshots,ssd,noatime 0 0
       - UUID=$ROOT_UUID /home btrfs subvol=@home,compress=zstd:3,ssd,autodefrag,noatime 0 0
       - UUID=$ROOT_UUID /data btrfs subvol=@data,compress=zstd:3,ssd,autodefrag,noatime 0 0
       - UUID=$ROOT_UUID /var btrfs subvol=@var,nodatacow,noatime 0 0
       - UUID=$ROOT_UUID /var/lib btrfs subvol=@var_lib,nodatacow,noatime 0 0
       - UUID=$ROOT_UUID /var/log btrfs subvol=@log,nodatacow,noatime 0 0
       - UUID=$ROOT_UUID /srv btrfs subvol=@srv,compress=zstd:3,ssd,noatime 0 0 
      - Add `tmpfs` entries at the end of the file:
        - tmpfs /tmp tmpfs defaults,noatime,nosuid,nodev,mode=1777 0 0
        - tmpfs /var/tmp tmpfs defaults,noatime,nosuid,nodev,mode=1777 0 0
      - #replace <PASTE_SWAP_OFFSET_HERE> with the actual numerical offset from echo $SWAP_OFFSET after running step 4e Manually insert the numerical offset into fstab for the swap file entry. Ensure the swap offset in /etc/fstab is the actual numerical value, not a shell substitution or variable.
      - cat /mnt/etc/fstab
 
**f) Check network**:
  - ping -c 3 archlinux.org
  - nmcli device wifi connect <SSID> password <password>  # If using Wi-Fi
  - Copy DNS into the new system so it can resolve mirrors
    - cp /etc/resolv.conf /mnt/etc/resolv.conf

## Step 5: **Install Arch Linux in the (/dev/nvme1n1)**
  - Mirrorlist Before pacstrap
    - pacman -Sy reflector  
    - reflector --latest 10 --sort rate --save /etc/pacman.d/mirrorlist  
  - Install base system:
    - pacstrap /mnt base base-devel linux linux-firmware mkinitcpio intel-ucode zsh nvidia-dkms nvidia-utils nvidia-settings opencl-nvidia cuda btrfs-progs sudo cryptsetup dosfstools efibootmgr networkmanager mesa libva-mesa-driver pipewire wireplumber pipewire-pulse pipewire-alsa pipewire-jack archlinux-keyring arch-install-scripts
  - Chroot into the system:
    - arch-chroot /mnt 
    - mv /crypto_keyfile /root/luks-keyfile
    - chmod 600 /root/luks-keyfile
  - Keyring initialization step
    - nano /etc/pacman.conf uncomment [multilib]
    - add **Include = /etc/pacman.d/mirrorlist** below the [core], [extra], [community], and [multilib] sections in /etc/pacman.conf
    - pacman-key --init
    - pacman-key --populate archlinux
    - pacman -Sy

## Step 6: **Set timezone, locale, and hostname**
  - systemctl enable --now fstrim.timer
  - ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
  - hwclock --systohc
  - echo 'en_US.UTF-8 UTF-8' > /etc/locale.gen
  - locale-gen
  - echo 'LANG=en_US.UTF-8' > /etc/locale.conf
  - echo 'thinkbook' > /etc/hostname
  - cat <<'EOF' > /etc/hosts
    - 127.0.0.1 localhost
    - ::1 localhost
    - 127.0.1.1 thinkbook.localdomain thinkbook
  - EOF

## Step 7: **Create User Account**
  - Set root password:
    - passwd 
  - Create a user with Zsh as the default shell:
    - useradd -m -G wheel,video,input,storage -s /usr/bin/zsh <username>
    - passwd <username>
  - Configure `sudo`:
    - sed -i '/^# %wheel ALL=(ALL:ALL) ALL/s/^# //' /etc/sudoers

## Step 8: **Set Up TPM and LUKS2 (re-enable Secure Boot in UEFI - replace sdb1 with sd#1 confirmed via lsblk)**

  **a) Install tpm2-tools:**
   - pacman -S --noconfirm tpm2-tools tpm2-tss

  **b) Bind LUKS2 Key to TPM:**
  - Enroll LUKS key:
   - systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+7 /dev/nvme1n1p2
   - cryptsetup luksDump /dev/nvme1n1p2 | grep -i tpm
   - sed -i 's/^BINARIES=(.*)/BINARIES=(\/usr\/lib\/systemd\/systemd-cryptsetup \/usr\/bin\/btrfs)/' /etc/mkinitcpio.conf
   - echo 'FILES=(/root/luks-keyfile)' >> /etc/mkinitcpio.conf
   - mkinitcpio -P
  - Update crypttab:
   - echo "cryptroot /dev/nvme1n1p2 /root/luks-keyfile luks,tpm2-device=auto,tpm2-pcrs=0+7" >> /etc/crypttab
   - tpm2_pcrread sha256:0,7 #Ensure PCRs 0 and 7 (firmware and Secure Boot state) are stable across reboots. If PCR values change unexpectedly, TPM unlocking may fail, requiring the LUKS passphrase. 
  - Back up keyfile to a secure USB:
   - lsblk 
   - mkfs.fat -F32 /dev/sdb1 **Replace sdb1 with USB partition confirmed via lsblk previously executed**
   - mkdir -p /mnt/usb
   - mount /dev/sdb1 /mnt/usb **Replace sdb1 with USB partition confirmed via lsblk previously executed**
   - cryptsetup luksHeaderBackup /dev/nvme1n1p2 --header-backup-file /mnt/usb/luks-header-backup
   - tpm2_exportpolicy --policy=/mnt/usb/tpm2-policy.bin --tpm-device=/dev/tpmrm0 --pcr-bank=sha256 --pcr-ids=0,7 /dev/nvme1n1p2
   - tpm2_createpolicy --policy-pcr -l sha256:0,7 -L /mnt/usb/tpm2-policy.bin **execute this one if the command above doesn't work, otherwise skip this step**
   - umount /mnt/usb

  **c) Enable Plymouth:**
   - Install and configure:
     - pacman -S --noconfirm plymouth
     - plymouth-set-default-theme -R bgrt
     - sed -i 's/HOOKS=(.*)/HOOKS=(base systemd autodetect modconf block plymouth sd-encrypt resume filesystems keyboard)/' /etc/mkinitcpio.conf
     - '# Ensure the order is: base systemd autodetect modconf block plymouth sd-encrypt resume filesystems. Incorrect order can cause Plymouth to fail or LUKS to prompt incorrectly. Ensure `plymouth` is before `sd-encrypt` in `/etc/mkinitcpio.conf` HOOKS and regenerate.
     - mkinitcpio -P  
     
## Step 9: **Configure systemd-boot with UKI**
  **a) Install systemd-boot:**
  - mount /dev/nvme1n1p1 /boot
  - bootctl --esp-path=/boot/EFI --no-variables install
       
  **b) Configure mkinitcpio for UKI:**
  - Edit `/etc/mkinitcpio.conf`:
    - #The Binaries is already completed in step 8b skip this action sed -i 's/^BINARIES=(.*)/BINARIES=(\/usr\/lib\/systemd\/systemd-cryptsetup \/usr\/bin\/btrfs)/' /etc/mkinitcpio.conf
    - sed -i 's/^MODULES=(.*)/MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm nvme)/' /etc/mkinitcpio.conf
    - sed -i 's/^HOOKS=(.*)/HOOKS=(base systemd autodetect modconf block plymouth sd-encrypt resume filesystems keyboard)/' /etc/mkinitcpio.conf
    - cat <<'EOF' > /etc/mkinitcpio.d/linux.preset # Do not append UKI_OUTPUT_PATH directly to /etc/mkinitcpio.conf.
      - default_uki="/boot/EFI/Linux/arch.efi"
      - default_options="rd.luks.uuid=$LUKS_UUID root=UUID=$ROOT_UUID resume=UUID=$ROOT_UUID resume_offset=$SWAP_OFFSET rw quiet nvidia-drm.modeset=1 splash intel_iommu=on iommu=pt pci=pcie_bus_perf,realloc mitigations=auto,nosmt slab_nomerge slub_debug=FZ init_on_alloc=1 init_on_free=1"
      - all_config="/etc/mkinitcpio.conf" 
    - EOF
    - mkinitcpio -P
  - Verify HOOKS order
    - grep HOOKS /etc/mkinitcpio.conf  # Should show block plymouth sd-encrypt resume filesystems 
   
  **c) Create Boot Entries:**
  - LUKS_UUID=$(cryptsetup luksUUID /dev/nvme1n1p2)
  - ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/cryptroot)
  - SWAP_OFFSET=$(cat /etc/swap_offset)
  - LUKS_UUID=$(blkid -s UUID -o value /dev/nvme1n1p2)
  - cat <<EOF > /boot/loader/entries/arch.conf # Should include resume=UUID=$ROOT_UUID -- Clarified that the swap file offset in /etc/fstab must be the literal numerical value from SWAP_OFFSET, not a command substitution. manually insert the numerical offset into fstab.
   - title Arch Linux
   - linux /EFI/Linux/arch.efi
   - options rd.luks.uuid=$LUKS_UUID root=UUID=$ROOT_UUID resume=UUID=$ROOT_UUID resume_offset=$SWAP_OFFSET rw quiet nvidia-drm.modeset=1 splash intel_iommu=on iommu=pt pci=pcie_bus_perf,realloc mitigations=auto,nosmt slab_nomerge slub_debug=FZ init_on_alloc=1 init_on_free=1
   - #Replace variable placeholder with actual number: $SWAP_OFFSET --> resume_offset=12345678  # Replace with actual output of btrfs inspect-internal map-swapfile
   - #if hotplug is not working consider looking some parameters i915.enable_psr=0 pci=nomsi or pci=nocrs or pcie_ports=native or pciehp.pciehp_force=1
   - EOF
  - cat <<'EOF' > /boot/loader/entries/windows.conf
   - title Windows
   - loader /EFI/Microsoft/Boot/bootmgfw.efi
   - EOF
  - cat <<EOF > /boot/loader/entries/arch-fallback.conf # fallback boot entry with minimal options for recovery
   - title Arch Linux (Fallback)
   - linux /EFI/Linux/arch.efi
   - options rd.luks.uuid=$LUKS_UUID root=UUID=$ROOT_UUID rw pci=pcie_bus_perf,realloc mitigations=auto,nosmt slab_nomerge slub_debug=FZ init_on_alloc=1 init_on_free=1
  - EOF

  
  **d) Set Boot Order:**
   - Pin Arch first:
     - BOOT_ARCH=$(efibootmgr | grep 'Arch Linux' | awk '{print $1}' | sed 's/Boot//;s/*//')
     - BOOT_WIN=$(efibootmgr | grep 'Windows' | awk '{print $1}' | sed 's/Boot//;s/*//')
     - efibootmgr --bootorder ${BOOT_ARCH},${BOOT_WIN}  # Ensure both Arch and Windows entries are listed
  
  **e) Create Fallback Bootloader:**
   - Create minimal UKI config: `/etc/mkinitcpio-minimal.conf` (copy `/etc/mkinitcpio.conf`, remove non-essential hooks).
     - cp /etc/mkinitcpio.conf /etc/mkinitcpio-minimal.conf
     - sed -i 's/HOOKS=(.*)/HOOKS=(base systemd autodetect modconf block sd-encrypt filesystems)/' /etc/mkinitcpio-minimal.conf
     - echo 'UKI_OUTPUT_PATH="/boot/EFI/Linux/arch-fallback.efi"' >> /etc/mkinitcpio-minimal.conf
   - Generate minimal UKI:
     - mkinitcpio -P -c /etc/mkinitcpio-minimal.conf
     - sbctl sign -s /boot/EFI/Linux/arch-fallback.efi
   - Create GRUB USB for recovery:
     - **Replace /dev/sdb1 with your USB partition confirmed via lsblk)**
     - lsblk
     - mkfs.fat -F32 -n RESCUE_USB /dev/sdb1
     - mkdir -p /mnt/usb
     - - **Replace /dev/sdb1 with your USB partition confirmed via lsblk)**
     - mount /dev/sdb1 /mnt/usb
     - pacman -Sy grub
     - grub-install --target=x86_64-efi --efi-directory=/mnt/usb --bootloader-id=RescueUSB
     - cp /root/luks-keyfile /mnt/usb/luks-keyfile
     - chmod 600 /mnt/usb/luks-keyfile 
     - cp /boot/vmlinuz-linux /mnt/usb/
     - cp /boot/initramfs-linux.img /mnt/usb/
     - LUKS_UUID=$(cryptsetup luksUUID /dev/nvme1n1p2)
     - ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/cryptroot) 
     - cat <<EOF > /mnt/usb/boot/grub/grub.cfg
       - set timeout=5
       - menuentry "Arch Linux Rescue" {linux /vmlinuz-linux cryptdevice=UUID=$LUKS_UUID:cryptroot cryptkey=UUID=$(blkid -s UUID -o value /dev/sdb1):fat32:/luks-keyfile root=UUID=$ROOT_UUID rw initrd /initramfs-linux.img}
       - EOF
     - sbctl sign -s /mnt/usb/EFI/BOOT/BOOTX64.EFI
    - After backing up to USB
     - umount /mnt/usb  # Replace with your USB mountpoint
     - shred -u /root/luks-keyfile
   
  **f) Add Pacman Hook for UKI Regeneration:**
   - mkdir -p /etc/pacman.d/hooks
   - cat << 'EOF' > /etc/pacman.d/hooks/90-mkinitcpio.hook
     - [Trigger]
     - Operation = Install
     - Operation = Upgrade
     - Type = Package
     - Target = linux
     - Target = linux-firmware
     - [Action]
     - Description = Regenerating UKI
     - When = PostTransaction
     - Exec = /usr/bin/mkinitcpio -P
   - EOF
  
## Step 10: **Configure Secure Boot** 

  **a) Install sbctl:**
   - pacman -S sbctl sbsign

  **b) Generate and Enroll MOKs:**
   - sbctl create-keys
   - sbctl enroll-keys --tpm-eventlog
   - **Reboot and enroll keys in UEFI firmware when prompted** After reboot, in chroot: 
   - sbctl sign -s /boot/EFI/Linux/arch.efi
   - sbctl sign -s /boot/EFI/Linux/arch-fallback.efi
   - sbctl sign -s /usr/lib/systemd/boot/efi/systemd-bootx64.efi
   - sbctl sign --all
   - sbctl status   # confirm “Secure Boot enabled; all OK”


  **c) Sign UKI and Nvidia Modules:**
   - KERNEL_VERSION=$(uname -r)
   - sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia.ko
   - sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia-modeset.ko
   - sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia-drm.ko
   - sbctl sign /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia-uvm.ko
   - #find /usr/lib/modules/$KERNEL_VERSION -name 'nvidia*.ko' -- if there are others nvidia modules sign them as well.
   - sbctl verify /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia*.ko

  **d) Automate Signing:**
   - cat <<'EOF' > /etc/pacman.d/hooks/99-secure-boot-signing.hook
     - [Trigger]
     - Operation = Install
     - Type = Package
     - Target = nvidia-dkms
     - Target = linux
     - Target = systemd
     - [Action]
     - Description = Signing kernel modules and UKI for Secure Boot...
     - When = PostTransaction
     - Exec = /usr/bin/bash -c "[ -x /usr/bin/sbctl ] && { KERNEL_VERSION=$(uname -r); sbctl sign $(find /usr/lib/modules/$KERNEL_VERSION/updates/dkms/nvidia/ -name "*.ko"); sbctl sign /boot/EFI/Linux/arch.efi; sbctl sign /usr/lib/systemd/boot/efi/systemd-bootx64.efi; sbctl sign /boot/EFI/Linux/arch-fallback.efi; }"
     - EOF

  **Reboot and verify Secure Boot is active.**

  **e) Verify:**
    - bootctl status | grep -i secure
    - sbctl status
    - sbctl verify /boot/EFI/Linux/arch.efi  # Should show as signed
     
## Step 11: **Install and Configure DE and Applications**

  **a) Install Gnome:**
   - pacman -S gnome
   - Install [Alacritty](https://github.com/alacritty/alacritty/blob/master/INSTALL.md) 
  - Reboot and Start gnome

  **b) Install the applications: (Review installations steps in each application instructions websites before installing it)**
  - Install `thinklmi` to verify BIOS settings:
   - pacman -S --needed thinklmi
  - Check BIOS settings: sudo thinklmi
  - After installing yay Configure to show PKGBUILD diffs
    - yay -Y --diffmenu --answerdiff=All || { echo "Failed to configure yay"; exit 1; }
   - Verify yay to show PKGBUILD diffs
    - yay -Pg | grep -E 'diffmenu|answerdiff'
  - Install core applications: #Use **--needed** with pacman and yay to avoid reinstalling existing packages. Review AUR PKGBUILDs
   - pacman -S --needed yay gnome-tweaks networkmanager bluez bluez-utils ufw apparmor tlp powertop cpupower upower systemd-timesyncd zsh snapper fapolicyd sshguard rkhunter lynis usbguard aide pacman-notifier mullvad-browser brave-browser tor-browser bitwarden helix zellij yazi blender krita gimp gcc gdb rustup python-pygobject git fwupd xdg-ninja libva-vdpau-driver libva-nvidia-driver zram-generator ripgrep fd eza gstreamer gst-plugins-good gst-plugins-bad gst-plugins-ugly ffmpeg gst-libav fprintd dnscrypt-proxy systeroid rage zoxide jaq atuin gitui glow delta tokei dua tealdeer fzf procs gping dog httpie bottom bandwhich gnome-bluetooth opensnitch
  - Install applications via Flatpak:
   - flatpak install flathub lollypop steam element-desktop Standard-Notes

 **c) Install Fonts**
  - yay -S --needed ttf-inter ttf-roboto noto-fonts ttf-ubuntu-font-family ttf-ibm-plex ttf-ubuntu-mono-nerd ttf-jetbrains-mono ttf-fira-code ttf-cascadia-code ttf-hack ttf-iosevka ttf-source-code-pro ttf-dejavu ttf-anonymous-pro catppuccin-cursors-mocha nerd-fonts-jetbrains-mono

 **d) Enable services:**
  - systemctl enable gdm bluetooth ufw auditd apparmor systemd-timesyncd tlp NetworkManager
  - After enabling all systemd services, run systemctl --failed to check for misconfigurations or missing dependencies.

 **e) Configure Flatseal for Flatpak apps:**
  - flatpak override --user --nofilesystem=host
  - flatpak override --user com.valvesoftware.Steam --device=dri  # Allow eGPU access

 **f) Configure Bubblejail for Alacritty and Conky:**
  - yay -S bubblejail
  - bubblejail create --profile generic-gui-app Alacritty
  - bubblejail create --profile generic-gui-app Conky
  - bubblejail config Alacritty --add-service wayland --add-service dri  # Allow eGPU access
  - bubblejail config Conky --add-service wayland --add-service dri
  - bubblejail run Alacritty -- env | grep -E 'WAYLAND|XDG_SESSION_TYPE'  # Verify Wayland

## Step 12: **Configure Power Management, Security and Privacy**

**a) Configure Power Management:**
   - systemctl enable --now tlp
   - systemctl mask power-profiles-daemon
   - systemctl disable power-profiles-daemon 
     - cat << 'EOF' > /etc/systemd/system/powertop.service
       - [Unit]
       - Description=Powertop Auto-Tune
       - After=multi-user.target suspend.target hibernate.target hybrid-sleep.target 
       - [Service]
       - Type=oneshot
       - ExecStart=/usr/bin/powertop --auto-tune
       - RemainAfterExit=yes
       - [Install]
       - WantedBy=multi-user.target suspend.target hibernate.target hybrid-sleep.target
     - EOF
     - systemctl enable --now powertop.service

 **b) Configure Wayland envars:**
   - cat << 'EOF' > /etc/environment
     - MOZ_ENABLE_WAYLAND=1
     - GDK_BACKEND=wayland
     - CLUTTER_BACKEND=wayland
     - QT_QPA_PLATFORM=wayland
     - SDL_VIDEODRIVER=wayland
   - EOF
   - cat << 'EOF' > ~/.config/wayland-nvidia-run
     - GBM_BACKEND=nvidia-drm
     - LIBVA_DRIVER_NAME=nvidia
   - EOF
   - chmod +x ~/.config/wayland-nvidia-run
   - gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"
   - gsettings set org.gnome.desktop.interface scaling-factor 1.25

 **c) Configure MAC randomization:**
   - mkdir -p /etc/NetworkManager/conf.d
   - cat << 'EOF' > /etc/NetworkManager/conf.d/00-macrandomize.conf
     - [device]
     - wifi.scan-rand-mac-address=yes
     - [connection]
     - wifi.cloned-mac-address=random
     - EOF
   - systemctl restart NetworkManager
   - nmcli connection down <connection_name> && nmcli connection up <connection_name>
   
 **d) Configure firewall:**
   - systemctl enable --now ufw
   - ufw allow ssh 
   - ufw default deny incoming
   - ufw default allow outgoing
   - ufw enable

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
     - * hard nproc 8192
     - EOF

 **h) Configure auditd:**
   - systemctl enable --now auditd
   - cat << 'EOF' > /etc/audit/rules.d/audit.rules
     - -w /etc/passwd -p wa -k passwd_changes
     - -w /etc/shadow -p wa -k shadow_changes
     - -a always,exit -F arch=b64 -S execve -k exec
     - EOF
  - systemctl restart auditd

 **i) Configure AppArmor with Nvidia Exceptions:**
   - apparmor_parser -r /etc/apparmor.d/*
   - aa-complain /usr/bin/nvidia-smi /usr/bin/nvidia-settings
   - cat <<'EOF' > /etc/apparmor.d/usr.bin.nvidia
     - /usr/bin/nvidia-smi r,
     - /usr/bin/nvidia-settings r,
     - /dev/nvidia* rw,
   - EOF
   - apparmor_parser -r /etc/apparmor.d/usr.bin.nvidia 

 **j) Configure dnscrypt-proxy:**
   - systemctl enable --now dnscrypt-proxy
   - nmcli connection modify <connection_name> ipv4.dns "127.0.0.1" ipv4.ignore-auto-dns yes #replace <connection_name> with your actual network connection (e.g., nmcli connection show to find it)
   - nmcli connection modify <connection_name> ipv6.dns "::1" ipv6.ignore-auto-dns yes
   - cat << 'EOF' > /etc/dnscrypt-proxy/dnscrypt-proxy.toml
     - server_names = ["quad9-dnscrypt-ip4-filter-pri", "adguard-dns-family", "nextdns-ultralow", "controld-dns", "mullvad-doh"]
     - listen_addresses = ["127.0.0.1:53", "[::1]:53"]
     - require_dnssec = true
     - require_nolog = true
     - require_nofilter = true
   - EOF
   - systemctl restart dnscrypt-proxy

  **k) Configure `usbguard` with GSConnect exception:**
   - usbguard generate-policy > /etc/usbguard/rules.conf
   - usbguard list-devices # Identify GSConnect device ID
   - usbguard allow-device <device-id>
   - systemctl enable --now usbguard

  **l) Enable `fapolicyd` with Nvidia exceptions, `sshguard`, `rkhunter`:**
   - pacman -S fapolicyd sshguard rkhunter
   - echo "/usr/bin/nvidia-smi trusted" >> /etc/fapolicyd/rules.d/90-custom.rules
   - echo "/usr/bin/nvidia-settings trusted" >> /etc/fapolicyd/rules.d/90-custom.rules
   - systemctl enable --now fapolicyd sshguard rkhunter
   - systemctl restart fapolicyd

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
   - systemctl enable lynis-audit.service
  
  **n) Configure AIDE:**
   - yay -S aide
   - aide --init
   - mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
   - systemctl enable --now aide-check.timer

  **o) Configure run0:**
   - yay -S run0
   
  **p) Configure sysctl hardening:**
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
     - kernel.dmesg_restrict=1
     - kernel.kptr_restrict=2
     - net.core.bpf_jit_harden=2 
   - EOF
   - sysctl -p /etc/sysctl.d/99-hardening.conf

  **q) Audit SUID binaries:**
   - find / -perm -4000 -type f -exec ls -l {} \; > /data/suid_audit.txt
   - cat /data/suid_audit.txt # Remove SUID from non-essential binaries
   - chmod u-s /usr/bin/ping
   - setcap cap_net_raw+ep /usr/bin/ping

  **r) Configure zram:**
   - cat << 'EOF' > /etc/systemd/zram-generator.conf
     - [zram0]
     - zram-size = 50%
     - compression-algorithm = zstd
   - EOF
   - systemctl enable --now systemd-zram-setup@zram0.service

## Step 13: **Configure eGPU (Nvidia and AMD)**
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
    - echo -e "GBM_BACKEND=nvidia-drm\n__GLX_VENDOR_LIBRARY_NAME=nvidia" | sudo tee -a /etc/environment
  - Create udev rules for hotplugging (The udev rule you've written uses loginctl terminate-user $USER, which will forcefully log you out and close all your applications every time you connect or disconnect the eGPU. Recommendation: Modern GNOME on Wayland has improved hot-plugging support. Your first step should be to test hot-plugging without any custom udev rules. It may already work. **Start with no rule, and only add one if you find it's absolutely necessary.**):
    - lspci -nn | grep -i nvidia  # Note the device ID, e.g., 0x1e84 
    - cat << EOF | sudo tee /etc/udev/rules.d/99-nvidia-egpu.rules # don't create this unless necessary for hotplug
      - - **Replace <device_id> below with Nvidia RTX 5070Ti device ID from lspci previouly executed**
      - ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{device}=="0x<device_id>", RUN+="/usr/bin/bash -c 'modprobe nvidia nvidia_modeset nvidia_uvm nvidia_drm; systemctl restart gdm'"
      - ACTION=="remove", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{device}=="0x<device_id>", RUN+="/usr/bin/bash -c 'modprobe -r nvidia_drm nvidia_uvm nvidia_modeset nvidia; systemctl restart gdm'"
    - EOF
    - udevadm control --reload-rules && udevadm trigger
    - echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia.conf
  - Enable PCIe hotplug:
    - echo "pciehp" | sudo tee /etc/modules-load.d/pciehp.conf
  - Check IOMMU groups for GPU passthrough
    - lspci -nnk  # Identify eGPU PCI ID
    - find /sys/kernel/iommu_groups/ -type l  # Check IOMMU group isolation. If passthrough needed, add vfio-pci.ids=<Nvidia/AMD device ID> to kernel parameters
  - Configure switcheroo-control
    - pacman -S switcheroo-control
    - systemctl enable --now switcheroo-control
  - Verify GPU switching:
    - DRI_PRIME=1 glxinfo | grep "OpenGL renderer"  # Should show AMD
    - prime-run glxinfo | grep "OpenGL renderer"    # Should show Nvidia
  - Install bolt for Thunderbolt/OCuLink device management
    - pacman -S bolt
    - systemctl enable --now bolt
    - boltctl list  # Authorize the OCuLink dock if listed
    - echo "always-auto-connect = true" | sudo tee -a /etc/boltd/boltd.conf
  - Verufy eGPU detection
    - lspci -nnk
    - DRI_PRIME=1 glxinfo | grep "OpenGL renderer"  # Test AMD
    - prime-run glxinfo | grep "OpenGL renderer"  # Test Nvidia  
   
## Step 14: **Configure Snapper**
  - Install Snapper:
    - yay -S snapper
  - Create global filter:
    - mkdir -p /etc/snapper/filters
    - echo -e "/home/.cache\n/tmp\n/run\n/.snapshots" | sudo tee /etc/snapper/filters/global-filter.txt
  - Create configurations:
    - snapper --config root create-config /
    - snapper --config home create-config /home
    - snapper --config data create-config /data
  - Edit `/etc/snapper/configs/root`:
    - cat << 'EOF' | sudo tee /etc/snapper/configs/root
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
      - cat << 'EOF' | sudo tee /etc/snapper/configs/home
      - TIMELINE_CREATE="yes"
      - TIMELINE_CLEANUP="yes"
      - TIMELINE_MIN_AGE="1800"
      - TIMELINE_LIMIT_HOURLY="0"
      - TIMELINE_LIMIT_DAILY="7"
      - TIMELINE_LIMIT_WEEKLY="4"
      - TIMELINE_LIMIT_MONTHLY="6"
      - TIMELINE_LIMIT_YEARLY="0"
      - SUBVOLUME="/home"
      - ALLOW_GROUPS=""
      - SYNC_ACL="no"
      - FILTER="/etc/snapper/filters/global-filter.txt"
      - EOF
      - cat << 'EOF' | sudo tee /etc/snapper/configs/data
      - TIMELINE_CREATE="yes"
      - TIMELINE_CLEANUP="yes"
      - TIMELINE_MIN_AGE="1800"
      - TIMELINE_LIMIT_HOURLY="0"
      - TIMELINE_LIMIT_DAILY="7"
      - TIMELINE_LIMIT_WEEKLY="4"
      - TIMELINE_LIMIT_MONTHLY="6"
      - TIMELINE_LIMIT_YEARLY="0"
      - SUBVOLUME="/data"
      - ALLOW_GROUPS=""
      - SYNC_ACL="no"
      - FILTER="/etc/snapper/filters/global-filter.txt"
      - EOF
    - Enable Snapper:
      - systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
    - Config permissions:
     - chmod 640 /etc/snapper/configs/* 
    - Verify configuration:
      - snapper --config root get-config
      - snapper --config home get-config
      - snapper --config data get-config
    - Test snapshot creation:
      - snapper --config root create --description "Initial test snapshot"
      - snapper --config home create --description "Initial test snapshot"
      - snapper --config data create --description "Initial test snapshot"
      - snapper list
        
     
 ## Step 15: **Configure Dotfiles**
  - Install chezmoi:
    - yay -S chezmoi
    - chezmoi init --apply
    - chezmoi add ~/.zshrc
    - chezmoi add -r ~/.config/gnome
    - dconf dump /org/gnome/ > ~/.config/gnome-settings.dconf
    - chezmoi add ~/.config/gnome-settings.dconf
    - chezmoi cd
    - git add . && git commit -m "Initial dotfiles"
   
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
   - filefrag -v /mnt/swap/swapfile  # Ensure no fragmentation
   - systemctl enable systemd-hibernate-resume.service
 - Test fwupd
   - fwupdmgr refresh
   - fwupdmgr update
 - Check for ThinkBook-specific quirks:
   - dmesg | grep -i "firmware\|wifi\|suspend\|battery" # Look for unusual warnings or errors related to hardware 
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
   - fio --name=write_test --filename=/data/fio_test --size=1G --rw=write
 - Verify Wayland
   - echo $XDG_SESSION_TYPE
 - Verify Security
   - auditctl -l
   - apparmor_status 
 - Test AUR builds with /tmp (no noexec)
   - yay --builddir ~/.cache/yay_build
 - Test Nvidia CUDA apps
   - prime-run blender
 - Verify Security Boot
   - mokutil --sb-state 

 ## Step 17: **Create Recovery Documentation**
  - Document UEFI password, LUKS passphrase, keyfile location, MOK password, and recovery steps in Bitwarden.
  - Create a recovery USB with Arch ISO, minimal UKI, and `systemd-cryptsetup`.
    - dd if=archlinux-<version>-x86_64.iso of=/dev/sdb bs=4M status=progress oflag=sync 
  - Back up LUKS header and SBCTL keys:
    - cryptsetup luksHeaderBackup /dev/nvme1n1p2 --header-backup-file /path/to/luks-header-backup
    - cp -r /etc/sbctl /path/to/backup/sbctl-keys

 ## Step 18: **Backup Strategy**
  - Local Snapshots:
    - Managed by Snapper for `@`, `@home`, `@data`, excluding `/var`, `/var/lib`, `/log`, `/tmp`, `/run`.
  - Offsite Snapshots:
    - To be refined savig the data in local server - check btrbk and restic
   
 ## Step 19: **Post-Installation Maintenance and Verification**
   **a) Regular System Updates:**
     - Always update your system regularly: `sudo pacman -Syu`
     - Check for AUR updates: `yay -Syu`

  **b) BTRFS Scrub:**
    - Schedule weekly or monthly BTRFS scrubs to check for data integrity issues:
      - `sudo btrfs scrub start /`
      - `sudo btrfs scrub status /`
      - Consider setting up a systemd timer for this (e.g., `btrfs-scrub@.timer` and `btrfs-scrub@.service` if provided by `btrfs-progs` or a custom one).

  **c) Remove Orphaned Packages:**
    - Periodically remove packages that are no longer required: `sudo pacman -Rns $(pacman -Qdtq)`

  **d) Review Snapper Snapshots:**
    - Regularly review your snapshots: `snapper list`
    - Manually delete old snapshots if needed (though `snapper-cleanup.timer` handles this): `snapper delete <snapshot_number>`

  **e) Check for SUID/SGID Changes (AIDE):**
    - `sudo aide --check` (After running `aide --init` and `mv` in initial setup)

  **f) Perform Security Audits:**
    - Run `lynis audit system` weekly/monthly (already scheduled by your timer).
    - Run `rkhunter --check` periodically.
    - Run `sudo usbguard generate-policy` if you add new USB devices and need to update rules.

  **g) Verify Firmware Updates:**
    - `fwupdmgr refresh` and `fwupdmgr update` periodically.

  **h) Check Systemd Journal for Errors:**
    - `journalctl -p 3 -xb` (errors from current boot)
    - `journalctl -p 3` (all errors)

  **i) Test AUR builds with /tmp (if noexec applied):**
    - If you encounter issues, consider configuring `yay` to use a different build directory (e.g., `yay --builddir ~/.cache/yay_build`) or temporarily removing `noexec` from `/tmp` for builds, then re-adding it. (This point is already in your Step 16, but it's good to reiterate it in maintenance as it's an ongoing consideration).
