+++
title = 'KVM虚拟机PCI直通+Looking Glass'
date = 2022-07-11T15:13:43+08:00
draft = false
math = true
tags = ["Linux","虚拟机","PCI直通"]
+++

## 名词解释
OVMF(Open Virtual Machine Firmware)用来提供虚拟环境。

QEMU：计算机模拟和虚拟机，可以模拟x86，arm等不同架构。

KVM为内置Linux内核的虚拟机管理程序，可与QEMU配合使用KVM虚拟化。

## 安装
### 前期准备

确认主板是否支持IOMMU(虚拟化，Intel Vt-d或AMD-Vi)，目前主流芯片组一般都可以开启，具体情况可以联系厂家客服。在BIOS中开启相关选项。

之后配置内核参数`intel_iommu=on`和`iommu=pt`，这一部分根据启动器的不同systemd-boot需要添加在`/boot/loader/entries/archlinux.conf`的`options`一行：
```
options root=UUID=0a3407de-014b-458b-b5c1-848e92a327a3 rw iommu=pt intel_iommu=on
```

GURB需要编辑`/etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt"
```
之后重新生成cfg:
```
# grub-mkconfig -o /boot/grub/grub.cfg
```
重启后检查dmesg：`dmesg | grep -e DMAR -e IOMMU`,如果有类似IOMMU:enable之类的输出则说明启用成功。
用下面的脚本查看显卡所在的IOMMU组：
```
#!/bin/bash
shopt -s nullglob
for d in /sys/kernel/iommu_groups/*/devices/*; do 
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```
找到显卡所在的IOMMU组，将组内**所有**设备最后面的id号记下，显卡所在组内可能还有一些音频设备也要一起隔离，在上文提到的内核参数中添加设备id:
```
vfio-pci.ids=10de:13c2,10de:0fbb
```
同时要保证vfio-pci提前加载,编辑`/etc/mkinitcpio.conf`,额外添加如下参数：
```
MODULES=(... vfio_pci vfio vfio_iommu_type1 vfio_virqfd ...)
HOOKS=(... modconf ...)
```

重新生成initramfs：
```
mkinitcpio -P
```
启动以后确认一下隔离状态`dmesg | grep -i vfio`,输出设备id即为正常。
### 虚拟系统安装
需要安装以下包：

```
pacman -S qemu libvirt ed2k-ovmf virtmanager

#同时需要将用户加入到libirt组中
gpasswd -a $USER libvirt
```
使用`virt-manager`管理虚拟机，创建img或qcow2镜像作为驱动盘，固件使用ovmf提供uefi环境，默认安装即可。

系统安装成功后可删除虚拟spice相关的显示/声音配置。在pci设备中添加显卡及鼠标键盘即可实现直通。
> 宿主机可以安装`virtio-win`驱动加速磁盘。虚拟机下载并安装[virtio-win](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/),下文的looking glass需要这个驱动。

## looking glass
可以在宿主机窗口下直接操控虚拟机，比较方便切换。
宿主机可通过aur安装，客户端需注意要相同版本，否则可能无法连通。
添加IVSHMEM：
```
<devices>
    ...
  <shmem name='looking-glass'>
    <model type='ivshmem-plain'/>
    <size unit='M'>32</size>
  </shmem>
</devices>
```
其中`size unit`需要根据屏幕像素计算：
$$\lceil \dfrac{Hight\times Width \times 8}{1024 \times 1024} + 10 \rceil = 2^n$$
计算结果向上取最接近2的n次幂，如1920x1080计算结果为25.8，最接近的数为32。

创建共享内存`/etc/tmpfiles.d/10-looking-glass.conf`:
```
f	/dev/shm/looking-glass	0660	$USER	kvm	
```
替换为自己的用户，运行：
```
# systemd-tmpfiles --create /etc/tmpfiles.d/10-looking-glass.conf
```
如果已经下载并安装[virtio-win](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/)的话可以在设备列表里看见IVSHMEM设备，运行`looking-glass-client`即可使用窥镜操控虚拟机。