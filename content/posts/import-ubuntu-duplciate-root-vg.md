---
title: 如何导入 Ubuntu 重复 root 卷组
date: 2020-07-19 23:30:00
categories: [Ubuntu,Linux,LVM]
---

## 场景

* 某云平台的 Ubuntu 虚拟机运行中突遇挂起，重启时挂起于某阶段，且无法进入恢复模式。
* 因该云平台限制或问题，无法通过 Ubuntu 安装 ISO 引导该虚拟机。
* 该虚拟机的 root 分区位于 LVM 卷组上。

当时，基本思路为挂载问题虚拟机的虚拟盘至新虚拟机，通过后者导入原虚拟盘的卷组，从而备份出重要数据。

然而，恰好该云平台仅有该虚拟机的原始模板，新实例的虚拟盘与旧盘：

* PV UUID 重复
* VG UUID 和名称重复

导致无法激活旧卷组。

## 办法

以下以 VirtualBox 安装 Ubuntu 20.04 虚拟机，克隆其系统盘为新虚拟盘，并将新盘加入该虚拟机来模拟场景。

### 启动系统

启动虚拟机将遇到 **重复 PV** 错误：

![Boot Ubuntu 20.04 duplicate root VG](/images/boot-focal-duplicate-root-vg.png)

进入 initramfs shell ，为了避免激活 root 卷组失败，暂时删除 `/dev/sdb` ：

```bash
echo 1 > /sys/block/sdb/device/delete
vgchange -ay
exit
```

将正常进入系统。

### 扫描 `/dev/sdb`

登录系统，切换 root 用户, 重新扫密 `/dev/sdb` ：

```bash
echo '- - -' > /sys/class/scsi_host/host1/scan
```

查看内核日志：

```bash
dmesg | tail
```

可发现 `/dev/sdb` 被加入系统：

```
[  747.671254] scsi 1:0:0:0: Direct-Access     ATA      VBOX HARDDISK    1.0  PQ: 0 ANSI: 5
[  747.674322] sd 1:0:0:0: [sdb] 20971520 512-byte logical blocks: (10.7 GB/10.0 GiB)
[  747.674345] sd 1:0:0:0: [sdb] Write Protect is off
[  747.674350] sd 1:0:0:0: [sdb] Mode Sense: 00 3a 00 00
[  747.674382] sd 1:0:0:0: [sdb] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[  747.678357] sd 1:0:0:0: Attached scsi generic sg1 type 0
[  747.681741]  sdb: sdb1 sdb2 < sdb5 >
[  747.681959] sd 1:0:0:0: [sdb] Attached SCSI disk
```

这里 `/dev/sda` 位于 SCSI host 0 ，而 `/dev/sdb` 位于 SCSI host 1 。实际环境中，若无法确定目标盘在第几个 host ，可全部扫描一遍：

```bash
for i in /sys/class/scsi_host/host*/scan; do
	echo '- - -' > $i
done
```

### 导入重复 VG

以新卷组名 `ubuntu` 执行 [vgimportclone](http://manpages.ubuntu.com/manpages/focal/man8/vgimportclone.8.html) 导入重复 VG ：

```bash
vgimportclone -n ubuntu /dev/sdb5
```

输出：

```
  WARNING: Not using device /dev/sdb5 for PV kyDNAt-ONdQ-9MIb-4tJO-w8kC-WnEY-xmJ2qj.
  WARNING: PV kyDNAt-ONdQ-9MIb-4tJO-w8kC-WnEY-xmJ2qj prefers device /dev/sda5 because device is used by LV.
```

`/dev/sdb5` 为新盘 VG 的 PV 。实际环境中，若 VG 存在多个 PV，则须全部作为该命令的参数。

### 激活新卷组

```bash
vgchange -ay
```

输出：

```
  2 logical volume(s) in volume group "ubuntu" now active
  2 logical volume(s) in volume group "vgfocal" now active
```
