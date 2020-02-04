---
layout: post
title: Mapping GPU framebuffer over PCI on ARM
---

This post covers mapping GPU framebuffer over PCI on ARM in Linux kernel.
Even though example is pretty exotic/outdated, it nevertheless is good for
illustrating some related concepts.

So let's say there's a GPU card with on-board RAM, connected over PCIe to ARM CPU.
Note: this example originates from archaic GPU, so no hardware acceleration
and fancy features of modern GPUs, just framebuffer.

PCI devices are accessible via memory, I/O (x86-only) and configuration spaces.
Since memory and I/O mapping needs to be explicitly established, process starts
with configuring device over configuration space. This usually happens during
low level initialization (e.g. BIOS for PC).
And by the time driver is initializing GPU, memory mapping is already configured
and available for the driver:
```C
	phys_addr_t video_base = pci_resource_start(pdev, 0);
	phys_addr_t mmio_base  = pci_resource_start(pdev, 1);
```
However addresses returned by `pci_resource_start` are physical, which means
that they need to be MMU mapped beforehand:
```C
	mmio_vbase = ioremap(mmio_base, mmio_size);
```

Additionally, memory region is explicitly claimed by the driver to ensure
exclusive access:
`request_mem_region(video_base, video_size, "GPU memory")`

Important point is that after configuration and mapping, GPU RAM is "visible"
to CPU, that is reading or writing instructions will physically go to GPU
through system interconnect. This gives several advantages:
1. CPU is offloaded.
2. Regular RAM can be used for other purposes.
3. GPU RAM is accessed as regular memory (convenience).

After all these steps, GPU memory can be accessed via as a regular memory,
in this case as a framebuffer containing picture data.
