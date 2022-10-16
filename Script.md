# Steps required to enable debian based image with qemu and uefi.

The script does the following:

1. Define where debian is going to be installed. It requires a disk device and mount point. Also defines network parameters.

```
# You will need a real Debian machine to perform cross debootstrap
# Debian on WSL does not work

# Define variables
TARGET=/dev/sda # Here goes your target device where load the image.
MOUNT_POINT=/mnt # Here goes mount point
IP_ADDRESS=192.168.100.145 # Here goes your device IP
GATEWAY=192.168.100.1 # Here goes your gateway IP
SSID=Your_SSID # Here goes your SSID considering that wifi is a choice
```

2. Prepares environment for debootstrap and qemu. This include packages, and preparing device with required partition formatting.

```
# Install dependency
apt update
apt install -y btrfs-progs dosfstools debootstrap qemu-user-static

# Set up partition table and file systems
dd if=/dev/zero of=${TARGET} bs=1M count=8
(echo g; echo n; echo ''; echo ''; echo '+512M'; echo t; echo 1; echo n; echo ''; echo ''; echo ''; echo w) | fdisk ${TARGET}
mkfs.fat -F32 ${TARGET}1
mkfs.btrfs -f ${TARGET}2

# Create subvolumes
mount -o compress=zstd ${TARGET}2 ${MOUNT_POINT}
btrfs subvolume create ${MOUNT_POINT}/@
btrfs subvolume create ${MOUNT_POINT}/@home
btrfs subvolume create ${MOUNT_POINT}/@root
umount ${MOUNT_POINT}

# Mount partitions
mount -o defaults,compress=zstd,subvol=@ ${TARGET}2 ${MOUNT_POINT}
mkdir -p ${MOUNT_POINT}/home
mount -o defaults,compress=zstd,subvol=@home ${TARGET}2 ${MOUNT_POINT}/home
mkdir -p ${MOUNT_POINT}/root
mount -o defaults,compress=zstd,subvol=@root ${TARGET}2 ${MOUNT_POINT}/root
mkdir -p ${MOUNT_POINT}/boot/efi
mount ${TARGET}1 ${MOUNT_POINT}/boot/efi

# First stage debootstrap
debootstrap --arch=arm64 --foreign --components=main,non-free,contrib --include=locales bullseye ${MOUNT_POINT} https://mirrors.tuna.tsinghua.edu.cn/debian

# Second stage debootstrap
cp /usr/bin/qemu-aarch64-static ${MOUNT_POINT}/usr/bin/
LANG=C.UTF-8 chroot ${MOUNT_POINT} /usr/bin/qemu-aarch64-static /bin/bash /debootstrap/debootstrap --second-stage
```

3. Prepare package repositories and disk rules for storage, include defining directories for copying uboot, dtb, network, kernel, hostname, and proxmox related weird stuffs.

```
# Set up fstab
echo "UUID=$(blkid -s UUID -o value ${TARGET}2) / btrfs defaults,compress=zstd,subvol=@ 0 0" >> ${MOUNT_POINT}/etc/fstab
echo "UUID=$(blkid -s UUID -o value ${TARGET}2) /home btrfs defaults,compress=zstd,subvol=@home 0 0" >> ${MOUNT_POINT}/etc/fstab
echo "UUID=$(blkid -s UUID -o value ${TARGET}2) /root btrfs defaults,compress=zstd,subvol=@root 0 0" >> ${MOUNT_POINT}/etc/fstab
echo "UUID=$(blkid -s UUID -o value ${TARGET}1) /boot/efi vfat defaults 1 2" >> ${MOUNT_POINT}/etc/fstab

# Update source.list
echo "deb [signed-by=/usr/share/keyrings/armbian-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/armbian/ bullseye main bullseye-desktop bullseye-utils" > ${MOUNT_POINT}/etc/apt/sources.list.d/armbian.list
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/proxmox-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/proxmox/debian bullseye pve-no-subscription"  > ${MOUNT_POINT}/etc/apt/sources.list.d/proxmox.list
echo "deb https://raw.githubusercontent.com/pimox/pimox7/master/ dev/" > ${MOUNT_POINT}/etc/apt/sources.list.d/pimox.list

curl https://raw.githubusercontent.com/pimox/pimox7/master/KEY.gpg | gpg --dearmour -o ${MOUNT_POINT}/usr/share/keyrings/pimox-archive-keyring.gpg
curl https://mirrors.tuna.tsinghua.edu.cn/armbian/armbian.key | gpg --dearmour -o ${MOUNT_POINT}/usr/share/keyrings/armbian-archive-keyring.gpg
curl https://enterprise.proxmox.com/debian/proxmox-release-bullseye.gpg | gpg --dearmour -o ${MOUNT_POINT}/usr/share/keyrings/proxmox-archive-keyring.gpg
printf "Package: *\no=Proxmox,a=stable,n=bullseye,l=Proxmox Debian Repository,c=pve-no-subscription,b=amd64\nPin-Priority: 250" > ${MOUNT_POINT}/etc/apt/preferences.d/pve-no-subscription
printf "Package: *\no=Armbian,a=bullseye,n=bullseye,l=Armbian\nPin-Priority: 250" > ${MOUNT_POINT}/etc/apt/preferences.d/armbian

# Add kernel hook for copying dtbs, which is required for U-Boot to boot grub
mkdir -p ${MOUNT_POINT}/boot/efi/dtb

cat << 'EOF' > ${MOUNT_POINT}/etc/kernel/postinst.d/copy-dtb
#!/bin/sh
set -e
version="$1"
if [ -e /usr/lib/linux-image-${version}/ ]
then
  echo Copying current dtb files to /boot/efi/dtb...
  cp -r /usr/lib/linux-image-${version}/* /boot/efi/dtb/
fi
EOF

chmod +x ${MOUNT_POINT}/etc/kernel/postinst.d/copy-dtb

# Configure network
cat << EOF > ${MOUNT_POINT}/etc/network/interfaces
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet static
address ${IP_ADDRESS}/24
gateway ${GATEWAY}
allow-hotplug wlan0
iface wlan0 inet dhcp
wpa-ssid ${SSID}
wpa-psk f62567f62499c41da515d88e7e877109e60a0cc320166d846450bc169e8b52d9
EOF

echo pve > ${MOUNT_POINT}/etc/hostname
echo ${IP_ADDRESS} pve pve.localdomaim >> ${MOUNT_POINT}/etc/hosts
```

4. Enabling chroot to enter filesystem created in step 2. From here, prepare timezone, language stuffs, and to install armbian and proxmox related packages via apt.

```
# Enter chroot
for i in /dev /dev/pts /proc /sys /run; do mount -B $i ${MOUNT_POINT}$i; done
LANG=es_CL.UTF-8 chroot ${MOUNT_POINT} /usr/bin/qemu-aarch64-static /bin/bash

# Define variables
USER=proxmox

# Configure locale and timezone
sed -i -e 's/# es_CL.UTF-8 UTF-8/es_CL.UTF-8 UTF-8/' /etc/locale.gen
echo 'LANG="es_CL.UTF-8"'> /etc/default/locale
dpkg-reconfigure -f noninteractive locales
update-locale LANG=es_CL.UTF-8
echo "America/Santiago" > /etc/timezone
ln -sf /usr/share/zoneinfo/America/Santiago /etc/localtime
dpkg-reconfigure -f noninteractive tzdata

# Install additional packages
apt update
apt install -y grub-efi linux-image-arm64 openssh-server sudo haveged zram-tools tmux bash-completion armbian-bsp-cli-orangepi4-lts dialog armbian-firmware linux-image-current-rockchip64 linux-dtb-current-rockchip64 linux-u-boot-orangepi4-lts-current wpasupplicant btrfs-progs
printf "SIZE=2048\nPRIORITY=100\nALGO=zstd\n" >> /etc/default/zramswap
systemctl enable ssh haveged zramswap

# Install pimox
apt install proxmox-ve pve-kernel-helper ifupdown2 openvswitch-switch
```

5. Finally, it supposed to create linux image (kernel, boot efi, etc) and weird stuffs inside chroot and then exit.

```
# Create a basic systemd-boot entry, please use `proxmox-boot-tool init` once inside the system to properly manage it
KERNEL=5.10.0-10-arm64
echo "root=UUID=$(blkid -o value -s UUID ${TARGET}2) rootflags=defaults,compress=zstd,subvol=@ console=ttyS2,1500000 console=tty1 consoleblank=0 loglevel=1 usb-storage.quirks=0x2537:0x1066:u,0x2537:0x1068:u cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory swapaccount=1" > /etc/kernel/cmdline
bootctl install
mkdir -p /boot/efi/EFI/proxmox/${KERNEL}
cp /boot/initrd.img-${KERNEL} /boot/vmlinuz-${KERNEL} /boot/efi/EFI/proxmox/${KERNEL}
cat << EOF > /boot/efi/loader/entries/proxmox-${KERNEL}.conf
title    Proxmox Virtual Environment
version  ${KERNEL}
options  $(cat /etc/kernel/cmdline)
linux    /EFI/proxmox/${KERNEL}/vmlinuz-${KERNEL}
initrd   /EFI/proxmox/${KERNEL}/initrd.img-${KERNEL}
EOF

# Clean up
apt-get clean

# Change root password
passwd

# Create new user for SSH login
useradd -m -s /bin/bash ${USER}
usermod -a -G sudo ${USER}
passwd ${USER}

# Exit chroot
exit
```

6. Set some configurations when rebooted board into uboot mode.

```
rm ${MOUNT_POINT}/usr/bin/qemu-aarch64-static
sync
umount -R ${MOUNT_POINT}

# After the installation, you need to upgrade your U-Boot to the latest mainline.
# Run following commands to load SPI overlay in U-Boot:
env default -f -a
setenv fdtover_addr_r 0x01fc0000
setenv fdtoverfile rockchip/overlay/rockchip-spi-jedec-nor.dtbo
saveenv
load mmc 0:1 $fdt_addr_r /dtb/$fdtfile
load mmc 0:1 $fdtover_addr_r /dtb/$fdtoverfile
load mmc 0:1 $kernel_addr_r /EFI/BOOT/BOOTAA64.EFI
fdt addr $fdt_addr_r
fdt resize 80000
fdt apply $fdtover_addr_r
bootefi $kernel_addr_r $fdt_addr_r
```
