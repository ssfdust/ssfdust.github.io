---
title: Vagrant环境搭建(libvirt)
date: '2022-05-16 09:12:28'
head: []
outline: deep
sidebar: false
prev: false
next: false
---



# Vagrant环境搭建(libvirt)

## 安装

```
# pacman -S vagrant
$ vagrant plugin install vagrant-libvirt
```

## 配置

### Libvirt

```
# usermod -a -G libvirt <user>
$ export LIBVIRT_DEFAULT_URI="qemu:///system"
$ let-env LIBVIRT_DEFAULT_URI = "qemu:///system"
$ virsh pool-define-as vagrant dir - - - - /usr/share/vagrant
$ virsh pool-build vagrant
$ virsh pool-start vagrant
$ virsh pool-autostart vagrant
```

#### Network

- 网络配置文件

```xml
<network ipv6='yes'>
  <name>vagrant-virbr2</name>
  <forward mode='nat'/>
  <bridge name='virbr2' stp='on' delay='0'/>
  <ip address='192.168.121.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.121.1' end='192.168.121.254'/>
    </dhcp>
  </ip>
  <dns>
    <forwarder addr="8.8.8.8"/>
  </dns>
</network>
```

- 使网络配置文件生效

```xml
$ virsh net-define network.xml
$ virsh net-start vagrant-virbr2
$ virsh net-autostart vagrant-virbr2
```

### Vagrant初始化

```
vagrant init xxxx
vagrant up --provider=libvirt
```

### Vagrant配置

#### provider配置

```
config.vm.synced_folder "./", "/vagrant", type: "nfs", nfs_udp: false, nfs_version: "4.2"
config.vm.provider :libvirt do |libvirt|
    # 存储池名称
    libvirt.storage_pool_name = "vagrant"
    # 使用系统接口
    libvirt.uri = "qemu:///system"
    # 默认网络修改
    libvirt.management_network_name = "vagrant-virbr2"
    # 阻止自定义网络被删除
    libvirt.management_network_keep = true
  end
```

## 问题记录

### Call to virDomainCreateWithFlags failed: Unable to get index for interface eth0

```xml
config.vm.network "public_network", ip: "192.168.1.89", bridge: "virtbr0", dev: "virtbr0", type: "bridge"
```

‍

### Kitty terminfo

```xml
infocmp -a xterm-kitty | vagrant ssh -c "tic -x -o \~/.terminfo /dev/stdin"
```

‍

### 透明代理没有网络

#### tcpdump

```
# tcpdump -i any host 47.117.137.199 and port 80
13:46:46.560770 vnet0 P   IP 192.168.121.232.48700 > 47.117.137.199.http: Flags [S], seq 2895886837, win 64240, options [mss 1460,sackOK,TS val 3837247923 ecr 0,nop,wscale 7], length 0
13:46:46.560770 virbr1 In  IP 192.168.121.232.48700 > 47.117.137.199.http: Flags [S], seq 2895886837, win 64240, options [mss 1460,sackOK,TS val 3837247923 ecr 0,nop,wscale 7], length 0
```

修改`prerouting`中的saddr，屏蔽掉网段

#### 修改DNS

```
virsh net-edit vagrant-libvirt
```

```
<network ipv6='yes'>
  <name>vagrant-libvirt</name>
  <uuid>90a8c64e-3da9-40bf-9df4-0e87580d1715</uuid>
  <forward mode='nat'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:6c:25:36'/>
  <dns>
    <forwarder addr='8.8.8.8'/>
  </dns>
  <ip address='192.168.121.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.121.1' end='192.168.121.254'/>
    </dhcp>
  </ip>
</network>
```

添加dns段，指定上游dns，需要将libvirtd添加到nft白名单

### 共享文件

**NFS挂载卡死**

```
config.vm.synced_folder "./", "/vagrant", type: "nfs", nfs_udp: false, nfs_version: "4.2"
```

调整nfs版本，4.2版本不支持`udp`​​协议，所以改为`false`

同时需要开启libvirtd区域防火墙的nfs服务

**防火墙未开启**

```
firewall-cmd --add-service=nfs --zone=libvirtd --permanent
```

### 自动在本地执行命令

通过trigger执行

```
```

- 小心返回值和错误输出，会导致任务异常

### 手动下载Box并载入

```
vagrant box add <boxname> /path/to/box
vagrant box add rockylinux/9 ~/Downloads/rockylinux9.box
```

### 9p挂载

#### 修改qemu默认为为root启动

用以解决虚拟机挂载非qemu用户目录时，没有权限的问题

```
sed -i -E 's/^[#]?\s?user\s?=\s?.*/user = "root"/' /etc/libvirt/qemu.conf
sed -i -E 's/^[#]?\s?group\s?=\s?.*/group = "root"/' /etc/libvirt/qemu.conf
```

#### 加载9p驱动(首次9p驱动加载失败后)

```
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

```
