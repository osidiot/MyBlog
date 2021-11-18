---
title: PCIe posted vs non-posted transactions
tags: [virtio, pcie, 体系结构]
---

### 问题背景

最近，在讨论如何在mlnx硬件卸载场景上支持virtio 0.95，因为virtio 0.95协议本身对硬件virtio直通不够友好，所以需要做些特殊的quirks处理。

面临最棘手的问题便是：virtio 0.95规范中对STATUS寄存器没有约定需同步等待，导致某些较老版本的virtio legacy驱动无法支持。举例来说：

```
static void vp_reset(struct virtio_device *vdev)
{
	struct virtio_pci_device *vp_dev = to_vp_device(vdev);
	/* 0 status means a reset. */
	iowrite8(0, vp_dev->ioaddr + VIRTIO_PCI_STATUS);
}
```

上述为Linux 2.6.39中virtio legacy驱动的实现，可见：**在reset设备时，将STATUS清零后直接返回**。

在QEMU模拟virtio场景下，此方式无问题，因为iowrite将引起陷出，最终由QEMU进行相应reset操作。

但是在硬件virtio VF直通场景下，cpu下发完reset请求后，VF硬件可能还没有处理完成，cpu便可继续执行后续操作，这便存在潜在风险：后续操作的正确性可能依赖于VF硬件已经reset完成，而彼时VF硬件可能尚在reset过程中。

&nbsp;

### 关键分析

#### cpu通过iowrite操作PCIe设备能否阻塞cpu？

在linux内核中，iowrite实际上有pio、mmio两种具体实现方式，首先需要明白我们场景中使用的是哪一种！

我们通过SR-IOV + vfio-pci直通方式将virtio VF直通给虚拟机，SR-IOV规范有个限制：**VF只支持memory BAR、不支持io BAR，因此对其访问只能是以mmio方式**。

说下答案：**不能!!!!**

如果mmio能够阻塞就好了，就能够在设备reset完成后再让cpu继续执行下去，我们就不会遇到上面的问题了。

那么，为什么不能呢？

mmio属于“Memory write”操作。在PCIe体系中，Memory write属于**Posted Transactions**，requester（此处即cpu）不需要等待completion TLP，可以立刻去干别的事情，类似导弹技术中的“发射后不管”。

&nbsp;

#### 较新的virtio legacy驱动是否有同步？

来看一下Linux 5.12中virtio legacy驱动的相关实现：

```
static void vp_reset(struct virtio_device *vdev)
{
	struct virtio_pci_device *vp_dev = to_vp_device(vdev);
	/* 0 status means a reset. */
	iowrite8(0, vp_dev->ioaddr + VIRTIO_PCI_STATUS);
	/* Flush out the status write, and flush in device writes,
	 * including MSi-X interrupts, if any. */
	ioread8(vp_dev->ioaddr + VIRTIO_PCI_STATUS);
	/* Flush pending VQ/configuration callbacks. */
	vp_synchronize_vectors(vdev);
}
```
