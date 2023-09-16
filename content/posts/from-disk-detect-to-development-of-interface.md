---
title: "从新硬盘检测到硬盘接口的发展"
date: 2023-09-16
draft: false
---

在工作中经常需要为虚拟机新增磁盘设备，为了在不 reboot 的情况下使用这些新磁盘，需要对新磁盘进行检测。而检测的方法是如下命令（方法来自 [Stack Exchange](https://unix.stackexchange.com/questions/404405/how-to-detect-new-hard-disk-attached-without-rebooting)）：

```bash
for host in /sys/class/scsi_host/*; do echo "- - -" | sudo tee $host/scan; ls /dev/sd* ; done
```

显然起作用的部分是

```bash
echo "- - -" > /sys/class/scsi_host/host*/scan
```

首先我们得需要了解硬盘的相关接口以及一些术语：

- SCSI（Small Computer System Interface）：包含了物理、电气、协议和命令等多个方面规范的连接和通讯的接口标准；SCSI 硬盘不集成控制器，控制器要么集成在主机要么通过主机的扩展槽来接入；
- IDE（Intergrated Drive Electronics）：字面上可以看得出来，表示的是控制器和盘体集成一体式的硬盘接口技术；但是后来在提到磁盘时，等价为了 ATA（PATA）；
- ATA（Advanced Technology Attachment）：IDE 的概念还是比较宽泛，在广泛应用后，其所使用的一些规范被归纳，应用在了
IBM AT 计算机上，被称之为 **AT**A 接口标准；而 IBM AT 的成功使得 ATA 成为了 IDE 技术的代表，在标准成熟后形成了 
ATA/ATAPI（ATA Packet Interface）标准；
- SATA（Serial ATA）：随着技术的发展，传输速率越来越快，传统并口 ATA（PATA，Parallel ATA）遇到了信号串扰和信号对齐等问题，限制了速率的进一步提高；因此在 2003 年 ATA 出现了串口的版本 SATA 1.0，并且在 2004 年立马推出了 SATA 2.0，提升了传输速率以及支持了**热插拔**，显然，在此之前 PATA 以及 SATA 1.0 是不支持热插拔的；2009 年推出了 SATA 3.0，每一个版本的传输速率都几乎翻倍；
- AHCI（Advanced Host Controller Interface）：引入于 2004 年，是一种驱动层面的数据传输协议规范；用于取代原有的 IDE（PATA）所使用的软件接口标准，以充分发挥 SATA 物理接口的实力。

> 所以从 IDE 宽泛的本义来看，SATA/PATA 这些都应该属于 IDE。

到此为止，这些接口和控制标准都是为 HDD 所制定的；但近些年来 SSD 高速发展，在初期只是通过 SATA 接口来使用 SSD，后来便很快碰到了瓶颈，需要专门为 SSD 制定新的接口及控制标准，以充分发挥 SSD 的性能优势。

- M.2：是一种物理标准，有 Socket 2 和 Socket 3 两种；Socket 2 可以走 SATA 3.0 或者 PCI-E 3.0x2（物理接口并不完全决定数据经过哪些通道到达内存或者 CPU，所以 M.2 这种物理接口是可以使用 SATA 3.0 的通道的；但是显然 SATA 3.0 接口是走的 SATA 3.0 的通道）；Socket 3 可以走 PCI-E 3.0x4；
- NVMe（None-Volatile Memory Express）：是一种通信接口和驱动程序，对标 AHCI，是为 SSD 数据走 PCI-E 通道设计和优化的；

> 说到这，可以看出 M.2 SSD 如果是 Socket 2 且走 SATA 3.0 通道，用的是 AHCI 协议；如果走 PCI-E 通道，就会用到 NVMe 协议。

从路径 `/sys/class/scsi_host/host*/scan` 可以看出，这应当是触发了 SCSI 适配器的扫描功能，但是结合上面的知识，会产生一个新的问题：为什么在对 `scsi_host` 下的控制器进行扫描，就能够检测新的硬盘呢，现在不都是 IDE 或者 AHCI 吗？

实际上 Linux 内核为了方便各种各种类型硬盘的接入，将它们抽象化为了统一的 SCSI 接口，便于上层应用的兼容，甚至 USB 磁盘设备也可能显示在这个目录下面，但最终还是由具体的驱动来与硬件进行交互的；显然，这也不是强制的，你可以专门针对 IDE 或者 SATA 进行开发。

对于每个 SCSI 设备，有确定的四元组来表示它：

- SCSI adapter number [host]
- channel number [bus]
- id number [target]
- lun [lun]

> LUN（Logical Unit Number）

因为 SCSI 控制器可能不止一个，因此需要 SCSI adapter number [host] 来指明；在前面提到的命令中，路径中的 host* 指明了这一点；每个控制器上可以有多条通道，这可以是物理上的划分，也可以是逻辑上的划分；然后每条通道上又可以连接多个 SCSI 设备内，因此需要 id 来区分；每个设备上又有多个 LUN，因此需要 lun 来进行区别，可见 SCSI 的最小控制单位是 LUN。

扫描指令参数 "- - -" 其实就是类似于 "* * \*"，表示在每个 channel，每个 target 上检测每个 lun，以发现新连接的磁盘设备。

> SATA 设备由于没有 LUN 的概念，因此在 SCSI 抽象中 LUN 只会有一个。
