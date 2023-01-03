

# ubuntu22配置NFS服务器

[toc]

## 在LVM2中创建一个LV

### 创建LV

查看vg：

```shell
root@k8s-node-3:/usr/include/nfs# vgs
  VG        #PV #LV #SN Attr   VSize    VFree
  sdavg       2   1   0 wz--n-   <1.82t    1.70t
  ubuntu-vg   1   1   0 wz--n- <464.26g <264.26g
```

在vg上创建lv：

```shell
root@k8s-node-3:/usr/include/nfs# lvcreate -L 30G -n nfslv sdavg
  Logical volume "nfslv" created.
```

查看lv：

```shell
root@k8s-node-3:/usr/include/nfs# lvs
  LV         VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  cltcloudlv sdavg     -wi-a----- 120.00g
  nfslv      sdavg     -wi-a-----  30.00g
  ubuntu-lv  ubuntu-vg -wi-ao---- 200.00g
```

这时候`lsblk -f`会发现还没有文件系统，需要给这个逻辑卷格式化，设置文件系统格式为ext4：

```shell
root@k8s-node-3:/usr/include/nfs# mkfs.ext4 /dev/mapper/sdavg-nfslv
mke2fs 1.45.5 (07-Jan-2020)
Discarding device blocks: done
Creating filesystem with 7864320 4k blocks and 1966080 inodes
Filesystem UUID: 46e5f060-c85f-4d1b-8f6e-e512fc1b13a8
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

`lsblk -f`查看块信息：

```shell
root@k8s-node-3:/usr/include/nfs# lsblk -f
NAME                      FSTYPE         LABEL   UUID                                   FSAVAIL FSUSE% MOUNTPOINT
loop0                     squashfs                                                            0   100% /snap/core18/2654
loop1                     squashfs                                                            0   100% /snap/core18/2667
loop2                     squashfs                                                            0   100% /snap/core20/1738
loop3                     squashfs                                                            0   100% /snap/snapd/17883
loop4                     squashfs                                                            0   100% /snap/lxd/24061
loop5                     squashfs                                                            0   100% /snap/lxd/23991
loop7                     squashfs                                                            0   100% /snap/core20/1778
sda
├─sda1                    LVM2_member            3Ph1tA-DPvf-rbF4-QKTl-ohXJ-sRBK-VcU3do
│ ├─sdavg-cltcloudlv      ext4           uploads 2e2e4373-7931-4d67-aa92-a61e1ceac185
│ └─sdavg-nfslv           ext4                   46e5f060-c85f-4d1b-8f6e-e512fc1b13a8
└─sda2                    LVM2_member            knFt64-7hcQ-nuAq-Wkld-GuwG-kk2R-TfXQjy
sdb                       ceph_bluestore
├─sdb2
└─sdb3
nvme0n1
├─nvme0n1p1               vfat                   BEB9-6AF2                               505.8M     1% /boot/efi
├─nvme0n1p2               ext4                   41b3a148-60f1-4153-808f-eaf5dd5067e3    700.7M    21% /boot
└─nvme0n1p3               LVM2_member            Hb3V1E-gjtl-mKvV-4gCs-qde2-5sLZ-u11dgr
  └─ubuntu--vg-ubuntu--lv ext4                   17867101-e6b4-4146-8c52-cdbe370a1cf9     32.9G    78% /
```

这样就完成了lv的创建（类似磁盘分区）。

### 挂载LV

创建一个目录作为挂载点：

```shell
root@k8s-node-3:/mnt# mkdir nfs1
root@k8s-node-3:/mnt# ls
nfs1  uploads
```

编辑`/etc/fstab`增加一条挂载：

```shell
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-pmHiy7I5xDSac44VrAj9HJQEZssd5o43Jbs28Scuq1P4a5caqBRZlM4WwnCLdq2A / ext4 defaults 0 0
# /boot was on /dev/nvme0n1p2 during curtin installation
/dev/disk/by-uuid/41b3a148-60f1-4153-808f-eaf5dd5067e3 /boot ext4 defaults 0 0
# /boot/efi was on /dev/nvme0n1p1 during curtin installation
/dev/disk/by-uuid/BEB9-6AF2 /boot/efi vfat defaults 0 0
#/swap.img      none    swap    sw      0       0

UUID=46e5f060-c85f-4d1b-8f6e-e512fc1b13a8 /mnt/nfs1 ext4 defaults 0 0

```

编辑完成后执行`mount -a`刷新。

查看`df -h`:

```shell
root@k8s-node-3:/mnt# df -h
...
/dev/mapper/sdavg-nfslv             30G   24K   28G   1% /mnt/nfs1
...
```

查看`mount`：

```shell
root@k8s-node-3:~# mount | grep nfslv
/dev/mapper/sdavg-nfslv on /mnt/nfs1 type ext4 (rw,relatime)
```



## 配置NFS服务器

运行`apt update & apt install nfs-kernel-server `安装服务器，查看进程状态：

```shell
root@k8s-node-3:~# systemctl status nfs-server.service
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset: enabled)
     Active: active (exited) since Tue 2023-01-03 03:32:45 UTC; 2h 13min ago
   Main PID: 739675 (code=exited, status=0/SUCCESS)
      Tasks: 0 (limit: 38305)
     Memory: 0B
     CGroup: /system.slice/nfs-server.service

```

**编辑`/etc/exports`配置共享目录：**

```shell
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
/mnt/nfs1  192.168.1.0/24(rw,sync,no_subtree_check)

```

编辑完成后执行`exportfs -a`刷出配置，重启后查看状态：

```shell
root@k8s-node-3:~# systemctl restart nfs-server
root@k8s-node-3:~# systemctl status nfs-server
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset: enabled)
     Active: active (exited) since Tue 2023-01-03 05:59:36 UTC; 6s ago
    Process: 1094903 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Process: 1094904 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
   Main PID: 1094904 (code=exited, status=0/SUCCESS)

```

查看nfs服务器上可用的共享目录：

```shell
root@k8s-node-3:~# exportfs -v
/mnt/nfs1       192.168.1.0/24(rw,wdelay,root_squash,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

```

修改共享目录的权限：

```shell
root@k8s-node-3:~# chown -R nobody:nogroup /mnt/nfs1
root@k8s-node-3:~# chmod 777 /mnt/nfs1
root@k8s-node-3:~# ll /mnt/
total 16
drwxr-xr-x  4 root   root    4096 Jan  3 03:29 ./
drwxr-xr-x 21 root   root    4096 Jan  2 09:28 ../
drwxrwxrwx  3 nobody nogroup 4096 Jan  3 03:27 nfs1/

```



## 配置NFS客户端

执行`apt update & apt install nfs-common -y`安装nfs客户端。

**挂载nfs服务器的共享目录到本地挂载点：**

```shell
root@k8s-node-2:~# mkdir /mnt/shared12
root@k8s-node-2:~# mount 192.168.1.12:/mnt/nfs1 /mnt/shared12
```

查看挂载点：

```shell
root@k8s-node-2:~# mount | grep nfs
192.168.1.12:/mnt/nfs1 on /mnt/shared12 type nfs4 (rw,relatime,vers=4.2,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.1.11,local_lock=none,addr=192.168.1.12)

```

## 测试

客户端写入：

```shell
root@k8s-node-2:~# echo "hi nfs" > /mnt/shared12/a.txt
root@k8s-node-2:~# cat /mnt/shared12/a.txt
hi nfs

```

服务器读取：

```shell
root@k8s-node-3:~# cat /mnt/nfs1/a.txt
hi nfs

```

服务器写入：

```shell
root@k8s-node-3:~# echo "hello client" >> /mnt/nfs1/a.txt
root@k8s-node-3:~# cat /mnt/nfs1/a.txt
hi nfs
hello client

```

客户端读取：

```shell
root@k8s-node-2:~# cat /mnt/shared12/a.txt
hi nfs
hello client

```



## 参考

- [lvm](https://www.cnblogs.com/sparkdev/p/10130934.html)

- [ubuntu22安装nfs](https://0xzx.com/2022071222502443886.html)