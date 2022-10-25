# pimox-kvm-orangepi4-lts
This is a little repo to store documentation about running Proxmox with KVM VMs support on an Rockchip 3399 (ARM64 big.LITTLE) SoC based Orange Pi 4 LTS SBC computer. This include info related with a script  which supposes to build debian-based debootstrap (in my case, on armbian host) from [here](https://gist.github.com/MakiseKurisu/c11f534568ffbc5d604797d67215daba), and a patch to apply in qemu server for running KVM with big.LITTLE CPUs.

## Background

I was able to run proxmox using [pimox manual installation](https://github.com/pimox/pimox7#manual-installation) on orangepi4-lts armbian ([debian bullseye 22.08](https://redirect.armbian.com/region/NA/orangepi4-lts/Bullseye_current)) image.

After configured vmbr0 interface with [Default configuration using bridge](https://pve.proxmox.com/wiki/Network_Configuration), I tried to build an HassOS arm64 image from [here](https://github.com/tteck/Proxmox#-pimox-haos-vm-). When trying to start, somehow it throwed me this error which is quite common:

```
kvm: kvm_init_vcpu: kvm_arch_init_vcpu failed (0): Invalid argument
TASK ERROR: start failed: QEMU exited with code 1
```

~~So the journey to enable KVM for big.LITTLE arch in Pimox begin here. BTW, pi 4 is not big.LITTLE, so that's why it does not have this problem.~~

~~Don't know for sure if KVM for big.LITTLE ARM requires having UEFI-compliant bootloader. In this case i assume that it has to be related with loading U-boot via SPI. Thinking maybe it's related that KVM can't select cpu cores from different clusters (RK3399 is non SMP as having two clusters with different cores).~~

~~A script chopped in pieces (for better self understanding but still need more work to done) is shown in [Script.md](https://github.com/gabrielribbae/pimox-kvm-orangepi4-lts/blob/main/Script.md), and serves as self-documentation while understanding the enabling process of KVM VMs. Don't take it as running that steps you will be able to fix this. Actually, i ran these steps and failed miserably because i think it is not supposed to run as script per se (i had some things related to chroot and env variables, also with repos not finding some packages).~~



Info gathered so far:

- https://github.com/pimox/qemu-server/pull/1. This is required to enable the functionality. The file is in /usr/share/perl5/PVE/QemuServer.pm.
- https://github.com/pimox/pimox7/issues/46
