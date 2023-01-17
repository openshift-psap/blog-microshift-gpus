# blog-microshift-gpus
In this blog we will demonstrate how to enable Nvidia GPUs on an x86 system running RHEL 8.7 and Microshift 4.12 developer preview. 
## Environment (Step 1)
Check your RHEL level and kernel level.
```bash
# cat /etc/redhat-release
Red Hat Enterprise Linux release 8.7 (Ootpa)
# uname -r 
4.18.0-425.3.1.el8.x86_64 
```
Nvidia drivers need to be installed for your specific kernel version. 
Precompiled kernel modules are availabe for the Nvidia drivers and have the advantage of being tested and validated against specific kernel versions. 
Look for your kernel version (in this example 4.18.0-425.3.1) in [Nvidia's precompiled kmod driver package table](https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/precompiled/); locate the matching Nvidia driver version in the table.
The driver version and kernel verison string must match exactly.  If your kernel version is not in the Nvidia precompiled kmod driver package table you can use the Nvidia Dynamic Kernel Module Support (DKMS) package to generate linux kernel modules for the Nvidia driver. To install the Nvidia drivers with RHEL using DKMS follow [Nvidia's instructions](https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html). 



## Verify the GPU installed on your system (Step 2)
```bash
# lspci -nnv |grep -i nvidia

example output:	
17:00.0 3D controller [0302]: NVIDIA Corporation GA100GL [A30 PCIe] [10de:20b7] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1532]
```

## Register you system with subscription manager (Step 3) 

```bash

# subscription-manager register
Registering to: subscription.rhsm.redhat.com:443/subscription
Username: <your username>
Password: <your password>
```

## Remove the Nouveau kernel Driver module (otherwise the Nvidia driver will fail to initialize the GPU), then install the Nvidia Driver and reboot (Step 3)
At this point in time the "latest" pre-compiled kernel modules for the Nvidia drivers support kernel version 4.18.0-372.32.1, as shown in [Nvidia's precompiled kmod driver package table](https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/precompiled/), so "525" is chosen here.  The life cycle is shown here.  First register your system with subscription-manager and install the precomplied modules. 
```bash

# echo 'blacklist nouveau' >> /etc/modprobe.d/blacklist.conf
# dnf config-manager --add-repo=https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
# dnf module install nvidia-driver:525 -y
# reboot
```

## Verify Nvidia Drivers are installed properly (Step 4) 
```bash
# nvidia-smi 

example output:

Fri Jan 13 14:29:53 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.60.13    Driver Version: 525.60.13    CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A30          Off  | 00000000:17:00.0 Off |                    0 |
| N/A   29C    P0    35W / 165W |      0MiB / 24576MiB |     25%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```


## Install Podman and crun, and verify that crun is the default OCI runtime (Step 5)
```bash
# dnf install -y crun
# dnf install -y podman
# cp /usr/share/containers/containers.conf /etc/containers/containers.conf

enable crun:
# sed -i 's/^#runtime = "crun"/runtime = "crun"/;' /etc/containers/containers.conf

disable runc:
sed -i 's/^runtime = "runc"/#runtime = "runc"/;' /etc/containers/containers.conf

check to make sure crun is uncommented 
# grep crun /etc/containers/containers.conf
```

## Install Nvidia Container Toolkit (Step 6)
Nvidia container toolkit supports [these distributions](https://nvidia.github.io/nvidia-docker/). Now we install Nvidia Container Toolkit for RHEL 8.7 which matches our version (shown in step 1).
```bash
# curl -s -L https://nvidia.github.io/nvidia-docker/rhel8.7/nvidia-docker.repo | tee /etc/yum.repos.d/nvidia-docker.repo
# dnf install nvidia-container-toolkit -y
```

## Add the nvidia-container SELinux policy to our machine (Step 7)
```bash
# curl -LO https://raw.githubusercontent.com/NVIDIA/dgx-selinux/master/bin/RHEL8/nvidia-container.pp
# semodule -i nvidia-container.pp
# nvidia-container-cli -k list | restorecon -v -f -
```



## Verify that the Nvidia Drivers and Podman are configured correctly (Step 8) 
```bash
# podman run --privileged -ti nvidia/cuda:11.0.3-base-ubuntu20.04 nvidia-smi


example output: 

Fri Jan 13 20:18:28 2023       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.60.13    Driver Version: 525.60.13    CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA A30          Off  | 00000000:17:00.0 Off |                    0 |
| N/A   30C    P0    32W / 165W |      0MiB / 24576MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+

```

## Install MicroShift (Step 9) 
Use the [4.12 release instructions to install MicroShift](https://54388--docspreview.netlify.app/microshift/latest/microshift_install/microshift-install-rpm.html). 
```bash

```
Refer to the Microshift Docs and follow steps in the [installation guide](https://microshift.io/docs/getting-started/). 



## Install Nvidia GPU Device Plugin (Step 10)
```bash

Selinux will prevent rootless containers from accessing the gpu without this setting.

# setsebool -P container_use_devices=true

Then install Nvidia GPU Device plugin
by coping the manifests to /etc/microshift/manifests

# oc get pods -n kube-system
NAME                                    READY   STATUS    RESTARTS   AGE
kube-flannel-ds-sxvz6                   1/1     Running   0          4d10h
nvidia-device-plugin-1670905057-xhjg9   1/1     Running   0          10h
# oc logs -n kube-system  nvidia-device-plugin-1670905057-xhjg9

2022/12/13 04:17:38 Retreiving plugins.
2022/12/13 04:17:38 Detected NVML platform: found NVML library
2022/12/13 04:17:38 Detected non-Tegra platform: /sys/devices/soc0/family file not found
2022/12/13 04:17:38 Starting GRPC server for 'nvidia.com/gpu'
2022/12/13 04:17:38 Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
2022/12/13 04:17:38 Registered device plugin for 'nvidia.com/gpu' with Kubelet

View the GPU in the node description.
# oc describe node | grep gpu
  nvidia.com/gpu:     1
  nvidia.com/gpu:     1
  nvidia.com/gpu     0           0

```
## Run a sample gpu test (Step 11)
```bash
# cat gpu.t
apiVersion: v1
kind: Pod
metadata:
  name: gpu-operand-test
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-vector-add
    image: "nvidia/samples:vectoradd-cuda11.2.1-ubi8"
    resources:
      limits:
         nvidia.com/gpu: 1
    securityContext:
      privileged: true


# oc create -f gpu.t

# oc logs -n default gpu-operand-test
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done


```
