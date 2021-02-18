## 平台

```
 ~]# rpm -q qemu-kvm
qemu-kvm-1.5.3-175.el7_9.1.x86_64
~]# rpm -q libvirt
libvirt-4.5.0-36.el7_9.3.x86_64
~]# cat /etc/redhat-release 
CentOS Linux release 7.9.2009 (Core)
```

- 查看CPU是否支持虚拟化技术 `~]# grep -E 'svm|vmx' /proc/cpuinfo`
- 宿主机开启路由转发 ：`net.ipv4.ip_forward = 1`
- 安装程序包：`~]# yum install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install virt-manager`
- 启动服务：`~]# yum install qemu-kvm libvirt libvirt-python libguestfs-tools virt-install`
- 启动服务：`~]#  systemctl enable libvirtd && systemctl start libvirtd`
- 验证 kvm 内核模块是否被装载

```
~]# lsmod |grep kvm
kvm_intel             183621  0
kvm                   586948  1 kvm_intel
irqbypass              13503  1 kvm
```

- 安装web控制台：`~]# yum install cockpit -y`
- 启动web控制台：`~]# systemctl  start  cockpit`

### 图形界面安装

- 调出控制台：`~]# virt-manager`

```
~]# virsh  list
 Id    名称                         状态
----------------------------------------------------
 2     vm1                            running
```

### 命令行模式安装

```
~]# ls /etc/libvirt/qemu
networks  vm1.xml
~]# ls /var/lib/libvirt/images/
vm1.qcow2
```

**命行模式使用模板xml文件和镜像qcow2格式的磁盘进行安装**

0. 关闭虚拟机 vm1 

```
~]# virsh  list --all
 Id    名称                         状态
----------------------------------------------------
 -     vm1                            关闭

```

1. 准备磁盘映像文件

```
~]# cp  /var/lib/libvirt/images/vm1.qcow2  /var/lib/libvirt/images/vm2.qcow2
```

2. 准备配置文件

```
~]# cp /etc/libvirt/qemu/vm1.xml  /etc/libvirt/qemu/vm2.xml
```

- 修改名称：<name>vm2</name>
- 修改uuid：<uuid>bc04e248-1fd2-4b1c-a379-b4c615d0a183</uuid>
- 修改磁盘镜像文件名：<source file='/var/lib/libvirt/images/vm2.qcow2'/>
- 修改 mac地址，只能修改后三段(6位)：<mac address='52:54:00:ed:1b:96'/>

3. 创建虚拟机

```
 ~]# virsh  define  /etc/libvirt/qemu/vm2.xml 
```

4. 重启 libvirtd

```
~]# systemctl  restart libvirtd
```

5. 启动虚拟机

```
~]# virsh start vm2
```

### 添加磁盘

1. 修改 xml 文件, 在 device 中添加 disk 

```
  <devices>
	...
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/vm2.qcow2'/>
      <target dev='hda' bus='ide'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <target dev='hdb' bus='ide'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <disk type='file' device='disk'>   // 此为新增加的磁盘
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/vm2-1.qcow2'/>
      <target dev='hdc' bus='ide'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    ...
  </devices>
```

2. 创建磁盘

```
~]# qemu-img  create -f qcow2 /var/lib/libvirt/images/vm2-1.qcow2 4G
```

3. 从新定义虚拟机

```
~]# virsh  define  /etc/libvirt/qemu/vm2.xml
~]# virsh  start vm2
```

### 存储池管理

1. 创建目录

```
 ~]# mkdir -pv /data/vmfs
```

2. 定义存储池和目录

```
 ~]# virsh pool-define-as vmdisk  --type dir --target /data/vmfs
```

3. 构建池

```
~]# virsh pool-build vmdisk 
```

4. 查看池

```
~]# virsh pool-list --all
 名称               状态     自动开始
-------------------------------------------
 default              活动     是       
 root                 活动     是       
 tmp                  活动     是       
 vmdisk               不活跃  否       
```

5. 激活并自动启动已经定义的存储池

```
~]# virsh pool-start vmdisk
~]# virsh pool-autostart vmdisk
```

6. 在存储池中创建虚拟机存储卷

```
~]# virsh vol-create-as vmdisk vol-vmdisk-01.qcow2 4G --format qcow2
```

#### 存储池管理命令

1.在存储池中删除虚拟机存储卷

```
~]# virsh  vol-delete --pool vmdisk vol-vmdisk-01.qcow2
```

2. 销毁存储池

```
~]# virsh pool-destroy vmdisk
```

4. 删除存储池定义的目录

```
~]# virsh pool-delete  vmdisk
```

5.取消存储池定义

```
~]# virsh pool-undefine vmdisk
```

#### 挂载磁盘到本地文件系统

1. 查看磁盘镜像的分区信息

```
~]# virt-df  -h -d vm1
文件系统                            大小 已用空间 可用空间 使用百分比%
vm1:/dev/sda1                            1014M       105M       909M   11%
vm1:/dev/centos/root                       17G       1.2G        16G    8%
~]# virt-filesystems -d vm1
/dev/sda1
/dev/centos/root
```

2. 挂在镜像分区

```
~]# guestmount  -d vm1  -m /dev/centos/root --rw /mnt
```

3. 卸载

```
~]# guestunmount /mnt/
```

### KVM 基本管理

### 启动、停止、销毁

### 虚拟机克隆

```
~]# virt-clone  -o vm1 -n vm3 --auto-clone 
~]# qemu-img  create -f qcow2 /var/lib/libvirt/images/vm4.qcow2 4G
```

#### 快照

raw 格式不支持快照

```
~]# virsh snapshot-create-as  vm2 vm2-snap
~]# virsh snapshot-list  vm2
 名称               生成时间              状态
------------------------------------------------------------
 vm2-snap             2021-01-01 09:13:24 +0800 shutoff
~]# virsh snapshot-revert   vm2 vm2-snap
```

raw 转化格式为 qcow2 

```
~]# qemu-img  convert -O qcow2 /var/lib/libvirt/images/vm1.img /var/lib/libvirt/images/vm-convert.qcow2
```

### 配置文件创建桥接网卡

```
~]# cd /etc/sysconfig/network-scripts/ && vim ifcfg-br0
TYPE=Bridge
NAME=br0
DEVICE=br0
ONBOOT="yes"
BOOTPROTO=static
IPADDR=192.168.40.132
GATEWAY=192.168.40.2
NETMASK=255.255.255.0
DNS1=114.114.114.114
network-scripts]# mv ifcfg-eth0{,.bak}
network-scripts]# vim ifcfg-eth0
DEVICE="eth0"
ONBOOT="yes"
BRIDGE=br0

network-scripts]# systemctl  restart network
network-scripts]# systemctl  restart libvirtd
```

- 删除桥接网卡

```
network-scripts]# mv ifcfg-br0 ifcfg-eth0 /tmp/ 
mv ifcfg-eth0.bak  ifcfg-eth0
systemctl restart network
systemctl  restart libvirtd
```

 
