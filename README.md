# blog-microshift-gpus
## Environment 
Check your kernel level 
```bash
#uname -r 
4.18.0-372.32.1.el8_6.x86_64 
```
Precompiled kernel modules for the Nvidia driver are compiled against specific kernel versions. 
Look for your kernel version (in this example 4.18.0-372.32.1) in [Nvidia's precompiled kmod driver package table](https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/precompiled/) and locate the matching Nvidia driver version in the table.
The driver version and kernel verison string must match exactly.  If your kernel version is not in the Nvidia precompiled kmod driver package table you can use the Nvidia DKMS packges or upgrade to a version that is in the table. 
