---
layout: post
title: "Linux 使用 LVM 来进行硬盘扩容"
date: 2014-12-26 14:56:45 +0800
comments: true
categories: 
---
# Linux 使用 LVM 进行硬盘扩容
## 前言
这阵子某个项目的服务器硬盘爆了，导致服务出现了异常，汗，现在用着 Google 和 阿里云 的云服务器，但发现两者都没有直接扩容的功能。所以就买了第二块硬盘，但第二块硬盘以后用满了怎么办？因此需要用到 LVM 来对硬盘进行动态扩容，不过并不能在云服务器的默认硬盘上，需要另外购置一块硬盘。

##什么是 LVM？

LVM 的全名是 Logical Volumn Manager，逻辑卷轴管理员，

一般来说，我们都会将某个硬盘映射到某个分区/目录，但总会有一天该硬盘会满掉，那怎么办呢？买第二块硬盘再把数据移过去？

LVM 就是为了解决这样的问题，它可以将多个硬盘不断的叠加起来，就像一块硬盘在使用一样。

其中有几个概念： PV VG LV PE

PV 指将硬盘转成 `Linux LVM` 格式

PE 是 LVM 的最小存储区块。

![](http://linux.vbird.org/linux_basic/0420quota/pe_vg.gif)

## 对硬盘进行分区
先切换到 root 用户
```ruby
sudo su -
```

查看系统已经识别的硬盘
```ruby
$ fdisk -l

# 显示主要信息，会发现有两块硬盘，sdb 则是我们刚刚购置的硬盘
Disk /dev/sda: 10.7 GB, 10737418240 bytes
Disk /dev/sdb: 75.2 GB, 75161927680 bytes

$ fdisk /dev/sdb      # 对硬盘进行操作

$  p(显示分区情况)
   n(新建分区)
   e(创建扩展分区)
   n -> l(创建逻辑分区)
   t(设置磁盘Hex code)——>#8e(LinuxLVM)——>#w(保存操作)
```

```ruby
$ fdisk /dev/sdb  
$ p

会发现：
 
  Device Boot      Start     End      Blocks      Id     System
    /dev/sdb1          1       2080     1048288+     5     Extended
    /dev/sdb5          1       2080     1048257      8e    Linux LVM

```

这里的逻辑分区 sdb5 已经成功的成为了 `Linux LVM`

```ruby
$ partprobe ( 让 kernel 重新读取磁盘分区表，即刻生效）
```

## 安装 LVM
```ruby
apt-get install lvm2
```

##创建 PV
```ruby
$ pvcreate /dev/sdb5      # 将此分区转换成为 PV

$ pvs    # 显示所有 PV 情况

  PV         VG   Fmt  Attr PSize  PFree
  /dev/sdb5       lvm2 a-   70.00g 70.00g
```

此时发现还没有一个VG

## 创建 VG
```ruby
$ vgcreate vg_data /dev/sdb5     # 这里的 vg_data 是自定义的

$ vgs   # 显示所有 VG 

  VG      #PV #LV #SN Attr   VSize  VFree
  vg_data   1   0   0 wz--n- 70.00g 70.00g
  
$ pvs  # 显示所有 PV

  PV         VG      Fmt  Attr PSize  PFree
  /dev/sdb5  vg_data lvm2 a-   70.00g 70.00g
  
  # 会发现 PV 下有个 VG 了  
```

## 创建 LV
```ruby
$ lvcreate -n lv_data -L 69.5G vg_data  # 另外的 0.5 G 需要用于其他用途

$ lvs  # 显示所有 LV

LV      VG      Attr   LSize  Origin Snap%  Move Log Copy%  Convert
  lv_data vg_data -wi-a- 69.50g
```

##格式化分区
```ruby
$ mkfs.ext4 /dev/vg_data/lv_data
$ fdisk -l
$ mount /dev/vg_data/lv_data /mnt/   # 挂到 mnt 下
```


## 以后如何动态扩容？
1. 用 fdisk 設定新的具有 8e system ID 的 partition
2. 利用 pvcreate 建置 PV
3. 利用 vgextend 將 PV 加入我們的 vg_data 
4. 用 lvresize 將新加入的 PV 內的 PE 加入 lv_data 中
5. 透過 resize2fs 將檔案系統的容量確實增加！

### 实际操作
```ruby
$ fdisk /dev/sdb10   => 其他动作参考上面
$ partprobe
$ pvcreate /dev/sdb10 
$ vgextend vg_data /dev/sdb10
$ vgdisplay # 显示 VG 的具体信息，会看到剩余的 PE ，假设为 90个
$ lvresize -l +90 /dev/vg_data/lv_data    # 放大 LV
```

这时 LVM 扩容了，但是档案系统还显示原先的大小

```ruby
resize2fs /dev/vg_data/lv_data  # 完整将 LV 容量扩充到整个 filesystem

# df /mnt/lvm   # 查看大小
```

##机子重启后自动挂载
```ruby
$blkid /dev/vg_data/lv_data  # 查看该设备的 UUID
$ vim /etc/fstab   # 编辑自动挂载的名单

=》UUID=1eb099f3-5332-43d7-a98d-6c1238572cd5 /home/deploy/red_mansions        ext4    defaults	0	0 

$ df -TH  # 查看下所有设备

Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/sda1      ext4       11G  2.7G  7.5G  27% /
udev           devtmpfs  2.0G  8.2k  2.0G   1% /dev
tmpfs          tmpfs     389M  242k  388M   1% /run
none           tmpfs     5.3M     0  5.3M   0% /run/lock
none           tmpfs     2.0G     0  2.0G   0% /run/shm

$  mount -a # 按照 fstab 文件重新挂载一遍
$  df -TH

Filesystem                  Type      Size  Used Avail Use% Mounted on
/dev/sda1                   ext4       11G  2.7G  7.5G  27% /
udev                        devtmpfs  2.0G  8.2k  2.0G   1% /dev
tmpfs                       tmpfs     389M  242k  388M   1% /run
none                        tmpfs     5.3M     0  5.3M   0% /run/lock
none                        tmpfs     2.0G     0  2.0G   0% /run/shm
/dev/mapper/vg_data-lv_data ext4       74G  7.5G   63G  11% /home/deploy/red_mansions

# 会发现设备已经挂载上去了。
```

## 给 设备 赋予授权
```ruby
$ chown -R deploy:deploy /home/deploy/  # 因为默认属于 root 权限
```
## PS

如果即将要  LV 化的硬盘原先有数据怎么办？ 需要先将数据移走，LV 化后再移进来。

## 参考

<http://linux.vbird.org/linux_basic/0420quota.php#lvm>
<http://www.ichiayi.com/wiki/tech/lvm>
<http://allen7111382.blog.51cto.com/202304/268562>
