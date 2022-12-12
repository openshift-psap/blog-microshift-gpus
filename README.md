# blog-microshift-gpus
## Environment 
Check your kernel level 
```bash
#uname -r 
4.18.0-372.32.1.el8_6.x86_64 
```
Nvidia drivers need to be installed for your specific kernel version. 
Precompiled kernel modules are availabe for the Nvidia drivers and have the advantage of being tested and validated against specific kernel versions. 
Look for your kernel version (in this example 4.18.0-372.32.1) in [Nvidia's precompiled kmod driver package table](https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/precompiled/); locate the matching Nvidia driver version in the table.
The driver version and kernel verison string must match exactly.  If your kernel version is not in the Nvidia precompiled kmod driver package table you can use the Nvidia Dynamic Kernel Module Support (DKMS) package to generate linux kernel modules for the Nvidia driver. To install the Nvidia drivers with RHEL using DKMS follow [Nvidia's instructions](https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html). 



## Verify the GPU installed on your system
```bash
# lspci -nnv |grep -i nvidia
17:00.0 3D controller [0302]: NVIDIA Corporation GA100GL [A30 PCIe] [10de:20b7] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1532]
```

## Remove the Nouveau kernel Driver module (otherwise the Nvidia driver will not load), then install the Nvidia Driver
At this point in time the "latest" pre-compiled kernel modules for the Nvidia driver support kernel version 4.18.0-372.32.1, as shown in [Nvidia's precompiled kmod driver package table](https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/precompiled/), so "latest" is chosen here.  
```bash
echo 'blacklist nouveau' >> /etc/modprobe.d/blacklist.conf
dnf config-manager --add-repo=https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
dnf module install nvidia-driver:latest -y
```

## Install Podman and crun, and verify that crun is the default OCI runtime
```bash
# dnf install -y crun
# dnf install -y podman
# cp /usr/share/containers/containers.conf /etc/containers/containers.conf
# sed -i 's/^# runtime = "crun"/runtime = "crun"/;' /etc/containers/containers.conf
```

## Install Nvidia-docker 
