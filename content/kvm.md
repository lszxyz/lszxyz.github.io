Title: 机哥amd小主机7840H kvm记录
Date: 2024-5-8 16:23
Modified: 024-5-8 16:23
Category: kvm
Tags: pelican, publishing
Authors: zero

#  机哥amd小主机7840H kvm记录

## 背景: 买不起服务器、出租房散热供电都不太行、之所以选择kvm、没有选择esxi 或者 proxmox 单纯想折腾一下、kvm里面跑esxi也是ok的嘛 在安装之前可以选坐的: bios修改功耗、加装硬盘、拆机注意后面排线、然后就可以开始愉快的all in boom之旅了

## 安装dedebian
下载iso镜像(我们这里直接选server版就好了)、写入u盘、usb引导启动、安装debian、配置源


## 安装kvm需要的包
```sh
apt install --no-install-recommends qemu-system libvirt-clients libvirt-daemon-system virt-manager dnsmasq-base bridge-utils firewalld
adduser <youruser> libvirt
virsh list --all

virsh --connect qemu:///system list --all
export LIBVIRT_DEFAULT_URI='qemu:///system'

virsh --help 
```

## 配置网络、存储
存储的话、依次分别是 disk >> pool >> vm/iso/template 
     
```sh
virsh pool-list --all
virsh pool-define-as --name ssd --type dir --target /data/pool/
virsh pool-start ssd
virsh pool-autostart ssd
virsh pool-info ssd
virsh --help | grep pool
```

网络 默认nat
* 桥接
* openvswitch

## vm/domain(kvm里面是叫domain、不是叫vm, why 具体可以参考这里)

vm的话、格式依次支持raw、qcow2、VMware format

常见几种操作:
- download iso >> create vm >> vnc connect and install 
- download cloud images >> import cloud image >> update config 
- terraform create vm 

### cloud image:
* https://gitlab.archlinux.org/archlinux/arch-boxes
* https://cdimage.debian.org/images/cloud/
* https://cloud-images.ubuntu.com/
* https://fedoraproject.org/cloud/

## 其他

## 参考链接
- https://wiki.debian.org/KVM
- https://linux-kvm.org/page/FAQ
- https://github.com/dmacvicar/terraform-provider-libvirt