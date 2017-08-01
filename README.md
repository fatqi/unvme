# unvme
Dear Eric,

I used a usersapce PCIe device driver in order to communicate with my PCIe storage device (NVMe SSD) bypass the OS kernel, it can reduce lost of SW overhead in kernel especially in block layer.

The userspace code is working well in X86 architechture, and the preconditions in X86 are: 1) CPU support VT-d (Virtualization Technology for Directed I/O), 2) Enable VFIO and IOMMU in kernel config. Now I'm trying to make the userspace code support ARM architechture, and then I faced above issue.

In my userspace code, I sent lots of IOCTL to VFIO reference kernel /Documentation/vfio.txt, and during the VFIO_SET_IOMMU ioctl, the kernel Oops BUG 0 occurred. The detail functions call trace is as below:

VFIO create // in my userspace code
dev->contfd = open("/dev/vfio/vfio", O_RDWR) // SUCCESS
...
ioctl(dev->groupfd, VFIO_GROUP_SET_CONTAINER, &dev->contfd) // SUCCESS
ioctl(dev->contfd, VFIO_SET_IOMMU, VFIO_TYPE1_IOMMU) // kernel Oops BUG 0 occurred, below is kernel functions call trace
----------------------------------// Above operations are in my userspace code ----------------------------------------------------
-> vfio_fops_unl_ioctl() // implement VFIO_SET_IOMMU ioctl
-> vfio_ioctl_set_iommu()
-> __vfio_container_attach_groups()
-> vfio_iommu_type1_attach_group() //kzalloc() for the domain success
-> iommu_attach_group()
-> iommu_group_do_attach_device()
-> arm_smmu_attach_dev() //in Arm-smmu.c "already attached to IOMMU domain" error occurred in this function and return -EEXIST
-> iommu_domain_free() // in Vfio_iommu_type1.c
-> arm_smmu_domain_free() //kfree() for the domain which kzalloc in vfio_iommu_type1_attach_group(), and kernel Oops BUG 0 occurred at the kfree() implementation. 

So that, I think the device should not attached to IOMMU domain in kernel initialization, it should be attached to SMMU domain in my userspace code.

And also I'm trying to test the Nvidia TX2 board SMMU via a ARM VFIO test code which I got from github, code: https://github.com/virtualopensystems/vfio-host-test Doc: http://www.virtualopensystems.com/en/solutions/guides/vfio-on-arm/
However, when I run this test code in TX2, all of VFIO platform devices already attached to IOMMU domain commentted by kernel dmesg, same issue with my userspace code. my code is similar with it, both are reference vfio.txt.

Below are the information which you comment in the last mail:
1.My PCIe device is PCIe NVMe SSD (PCIe*4lane)
2.Before my assignment (just after the TX2 board booting), the iommu_groups information as below:

nvidia@tegra-ubuntu:~$ ls /sys/kernel/iommu_groups/
0  10  12  14  16  18  2   21  23  25  27  29  30  32  34  36  38  4   41  43  45  47  49  50  52  54  6  8
1  11  13  15  17  19  20  22  24  26  28  3   31  33  35  37  39  40  42  44  46  48  5   51  53  55  7  9

nvidia@tegra-ubuntu:~$ dmesg | grep "to group"
[    0.725815] iommu: Adding device 3460000.sdhci to group 0
[    0.732077] iommu: Adding device 3400000.sdhci to group 1
[    0.740813] iommu: Adding device 3507000.ahci-sata to group 2
[    0.747217] iommu: Adding device 3160000.i2c to group 3
[    0.753003] iommu: Adding device c240000.i2c to group 4
[    0.758723] iommu: Adding device 3180000.i2c to group 5
[    0.764443] iommu: Adding device 3190000.i2c to group 6
[    0.770246] iommu: Adding device 31b0000.i2c to group 7
[    0.776009] iommu: Adding device 31c0000.i2c to group 8
[    0.781742] iommu: Adding device c250000.i2c to group 9
[    0.787467] iommu: Adding device 31e0000.i2c to group 10
[    0.794748] iommu: Adding device 3210000.spi to group 11
[    0.800595] iommu: Adding device c260000.spi to group 12
[    0.806455] iommu: Adding device 3240000.spi to group 13
[    0.812680] iommu: Adding device 3100000.serial to group 14
[    0.818779] iommu: Adding device 3110000.serial to group 15
[    0.824888] iommu: Adding device 3130000.serial to group 16
[    0.831651] iommu: Adding device 2490000.ether_qos to group 17
[    0.838135] iommu: Adding device b000000.rtcpu to group 18
[    0.847967] iommu: Adding device smmu_test to group 19
[    0.920539] iommu: Adding device 3530000.xhci to group 20
[    0.926480] iommu: Adding device 3550000.xudc to group 21
[    0.946721] iommu: Adding device 13e10000.host1x to group 22
[    0.952907] iommu: Adding device 13e10000.host1x:ctx0 to group 23
[    0.959478] iommu: Adding device 13e10000.host1x:ctx1 to group 24
[    0.966033] iommu: Adding device 13e10000.host1x:ctx2 to group 25
[    0.972618] iommu: Adding device 13e10000.host1x:ctx3 to group 26
[    0.979324] iommu: Adding device 13e10000.host1x:ctx4 to group 27
[    0.985887] iommu: Adding device 13e10000.host1x:ctx5 to group 28
[    0.992445] iommu: Adding device 13e10000.host1x:ctx6 to group 29
[    0.999049] iommu: Adding device 13e10000.host1x:ctx7 to group 30
[    1.005818] iommu: Adding device 150c0000.nvcsi to group 31
[    1.012071] iommu: Adding device 15700000.vi to group 32
[    1.017916] iommu: Adding device 15600000.isp to group 33
[    1.023983] iommu: Adding device 15210000.nvdisplay to group 34
[    1.030398] iommu: Adding device 15340000.vic to group 35
[    1.036229] iommu: Adding device 154c0000.nvenc to group 36
[    1.042232] iommu: Adding device 15480000.nvdec to group 37
[    1.048461] iommu: Adding device 15380000.nvjpg to group 38
[    1.054499] iommu: Adding device 15500000.tsec to group 39
[    1.060419] iommu: Adding device 15100000.tsecb to group 40
[    1.067096] iommu: Adding device 15810000.se to group 41
[    1.072831] iommu: Adding device 15820000.se to group 42
[    1.078596] iommu: Adding device 15830000.se to group 43
[    1.084320] iommu: Adding device 15840000.se to group 44
[    1.090312] iommu: Adding device 17000000.gp10b to group 45
[    1.104509] iommu: Adding device d000000.bpmp to group 46
[    1.123933] iommu: Adding device 2600000.dma to group 47
[    1.172703] iommu: Adding device 10003000.pcie-controller to group 48
[    1.179653] iommu: Adding device sound to group 49
[    1.186081] iommu: Adding device 3510000.hda to group 50
[    1.192600] iommu: Adding device adsp_audio to group 51
[    1.199200] iommu: Adding device 2993000.adsp to group 52
[    1.210307] iommu: Adding device c160000.aon to group 53
[   14.790917] iommu: Adding device 0000:00:01.0 to group 54
[   14.791325] iommu: Adding device 0000:01:00.0 to group 55

Device 0000:01:00.0 in above is my PCIe NVMe SSD, from this dmesg, we can know all of devices were attached to IOMMU domian in kernel initialization. So, I think that due to it already attached to IOMMU domain, so it cannot be assigned to SMMU domain in my userspace code. I'm not sure but looks like that from the kernel code debug (pls see above detail functions call trace) 

3.My test case is based on Micron UNVMe Driver https://github.com/MicronSSD/unvme, I did lots of modification in order to adapt my NVMe SSD firmware, but the code of VFIO part are same, and also same with above vfio-host-test project main() function, you can check these two code, and also can build it to test. You need comment the "asm rdtsc" in rdtsc.h in unvme project at first, and then it can build success in ARM architechture.

Now all of my jobs are suspended by this issue, I really need your help, thank you! I appreciate your help very much!

Now all of my jobs are suspended by this issue, I really need Nvidia's help, thank you!
