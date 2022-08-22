### IXY 用户态网卡驱动原理

#### 非virtIO网卡

网卡接收网络数据包, 使用**DMA(Direct memory access, IO设备直接访问物理地址)**模式直接写到内存。

- UIO与VFIO

- - UIO
Userspace I/O, Linux kernel提供的接口。用户空间程序能通过访问`/sys/bus/pci/devices/pci_addr/` 目录直接访问PCIe设备, 对设备有完全的控制.

- - VFIO
Virtual function I/O, 用户访问PCIe设备需要先把该设备绑定到一个通用的vfio-pci驱动上, 然后通过`ioctl`来与PCIe设备交互. 只有VFIO支持**interrupts**模式

- 网卡(PICe设备)控制
驱动与网卡PCIe设备通信有两种模式: **MMIO(Memory-mapped I/O)**, **PMIO(Port-mapped I/O)**。 

- - **PMIO**
驱动通过特殊类别的CPU指令来向PCI设备完成I/O, 在PCIe中已废弃.

- - **MMIO**
驱动通过mmap映射PCIe设备的BARs(Base Address **Registers**) 到内存中(这里需要的是BAR0), 然后通过获取或修改这些不同的**寄存器**的值来获得或传递不同的控制信息.
UIO模式下, 直接 `mmap(/sys/bus/pci/devices/pci_addr/resource0, ...`, 即把BAR0的寄存器mmap到内存中了。
VFIO模式下, 需先获取PCIe设备IOMMU组ID, 然后通过组ID和PCIe地址获取PCIe设备的fd(`vfio_fd`), 然后通过`ioctl(vfio_fd, VFIO_DEVICE_GET_REGION_INFO, &region_info);`获取mmap的地址和范围, 最后通过 `mmap(NULL, region_info.size, PROT_READ | PROT_WRITE, MAP_SHARED, vfio_fd, region_info.offset)`将BAR0的寄存器mmap到内存中。

Driver mmap PCIe的BAR0之后, 就能通过设置PCIe的**寄存器**来控制PCIe设备。包括设置网卡的`rx_queue和tx_queue`、开启设备的`DMA`模式、设置网卡的`promiscuous`模式...


- 网卡队列
  网卡(port)有最大支持的rx和tx队列数量, 通过获取对应的**寄存器**值(**MMIO**已映射到内存中)查看, 不同网卡寄存器名字不同。
  
  驱动首先通过大页内存分配**rx_queue**和**tx_queue**(这里的rx、tx queue只是一个**描述符队列**,每一个元素都记录着**一个**实际存储数据包的**物理地址**)。 然后把**rx_queue**和**tx_queue**的物理地址直接发给设备(通过设置**寄存器**值, 所有信息传递给设备都是通过**寄存器**)。设备通过DMA模式就能通过它们的物理地址直接访问的**rx_queue**和**tx_queue**。
  
  为何使用大页来分配rx、tx queue, 是因为DMA访问的是物理地址, linux操作系统有内存迁移技术, 使用传统的malloc分配出来的内存在任何时候它的物理地址都可能变化, 而大页内存分配出的内存物理地址是不变的。
  
  如上所说, rx、tx queue只是一个描述符队列, 所以还需要为rx、tx queue中**每个**元素分配**一个**对应的数据包存储空间(pkt_buf, 不要求连续, 即不要求rx[i]对应pkt_buf[i], 因为每个pkt_buf都可独立分配), 也使用大页分配, rx、tx queue中每个元素记录对应的pkt_buf的物理地址。
  
  网卡从rx中取一个可用元素后, 会将包写到rx[i]对应的pkt_buf上, 通过DMA直接访问pkt_buf的物理地址。
    从tx中取一个可用元素后, 会将tx[i]对应的pkt_buf发送出去.

  **\[  \]\[  \]\[  \]\[  \]  			--- > 		\[        \]\[        \]\[        \]\[        \]\[        \]**
  `rx/tx queue` 						`pkt_buf` 

  都是通过大页分配, 驱动给上层应用提供数据包即直接返回pkt_buffer[i]，内存位于大页中, 直接使用.


#### virtIO网卡

以下均是对驱动描述

virtIO目前是使用UIO模式。

virtIO通过`/sys/bus/pci/devices/pci_addr/config` 开启DMA模式。

virtIO `mmap(/sys/bus/pci/devices/pci_addr/resource0,...)`MMIO 映射寄存器, 获取设备信息。

virtIO网卡有三个队列(Virtqueue): receive queue、transmit queue、command queue.
每个Virtqueue由descriptor table和两个rings(the available and used rings)组成

分配Virtqueue方式与**非virtIO**的**rx tx queue**大约相同(具体看ixy代码`virtio.c:virtio_legacy_setup_rx_queue`), 也都是**描述符队列**, 有对应的pkf_buf地址。

virtIO设置promiscuous模式是通过**command queue**和**寄存器**。

