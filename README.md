# pimox-kvm-orangepi4-lts
This is a little repo to store documentation about running Proxmox (which use KVM for VMs) on an Rockchip 3399 (ARM64 big.LITTLE) SoC based Orange Pi 4 LTS SBC computer, through applying a patch in qemu server for running KVM with big.LITTLE CPUs. KVM needs QEMU (emulator) for full functionality as a hypervisor. QEMU is self-contained and KVM is actually a Linux kernel module for exploiting VT extensions that acts as a driver for physical CPU capabilities.

## Steps to enable proxmox on orangepi4

### 1. Preparing an microSD card with armbian

You can download BalenaEtcher to flash armbian on orangepi4-lts. I used ([debian bullseye 22.08](https://redirect.armbian.com/region/NA/orangepi4-lts/Bullseye_current)) image. I'll recommend using nand-sata-install to eMMC after this as it has more i/o speeds. I had encountered problems trying to flash to eMMC before installing proxmox.

### 2. Perform pimox manual installation instructions

I was able to run proxmox using [pimox manual installation](https://github.com/pimox/pimox7#manual-installation).

#### IMPORTANT

At the moment of this guide being written, on this setup you'll may encounter errors during install, specifically related not be able to use ZFS and Ceph functionalities out of the box, as there are some incompability issues with package versions and kernel from 5.15 and above. You can try some [workaround like this](https://github.com/pimox/pimox7/issues/66#issuecomment-1186114928) to enable them but in my case using 5.15.69-rockchip64 kernel image installed proxmox without them, which considering this as a test environment is fine.

### 3. Basic configuration

After configured vmbr0 interface with [Default configuration using bridge](https://pve.proxmox.com/wiki/Network_Configuration), I tried to build an HassOS arm64 image from [here](https://github.com/tteck/Proxmox#-pimox-haos-vm-). 

### 4. Enabling a example VM

In this case, i've used a [helper templates given from this repo](https://github.com/tteck/Proxmox), which has very cool tools. You should run these inside proxmox shell to work properly.


### 5. Patch KVM error related to big.LITTLE architecture

Normally, when trying to start VMs, on big.LITTLE ARM arch it throws this error which is quite common:


```
kvm: kvm_init_vcpu: kvm_arch_init_vcpu failed (0): Invalid argument
TASK ERROR: start failed: QEMU exited with code 1
```

So the journey to enable KVM for big.LITTLE arch in Pimox begin here (BTW, raspberry pi 4 is not big.LITTLE, so that's why it does not have this problem).

~~Don't know for sure if KVM for big.LITTLE ARM requires having UEFI-compliant bootloader. In this case i assume that it has to be related with loading U-boot via SPI. Thinking maybe it's related that KVM can't select cpu cores from different clusters (RK3399 is non SMP as having two clusters with different cores).~~

You'll need to [apply this patch](https://github.com/pimox/qemu-server/pull/1). ATM i think it's not applied upstream but it's still as pending commit. This is required to enable the functionality. The file is in /usr/share/perl5/PVE/QemuServer.pm. You can find another issue [commenting this fix](https://github.com/pimox/pimox7/issues/46). 

Qemuserver file has changed since this patch was pushed, but you could see in the commit that it is required to insert lines of codes on two different parts of the program. For this, you can reference where to apply it manually (e.g. using nano) searching these lines:

On the `cpu_taskset` definition, find:
```
description => "Configure a VirtIO-based Random Number Generator.",
```
and apply it below.

For the `if (defined($conf->{cpu_taskset}))` definition, find:
```
push @$cmd, $kvm_binary;
```
and apply it above

With that applied, you'll need to write on your vm.conf stored in `/etc/pve/qemu-server` these parameters:

```
cores: 4
cpu: host
cpu_taskset: 0-3
```
In this case, worth noticing that for RK3399, cores from 0-3 are A53 Cortex , while 4-5 are A72 Cortex cores.

### 6. Run VM

With that, you should be able to run proxmox VMs inside an RK3399 SoC Orangepi4-lts :)
