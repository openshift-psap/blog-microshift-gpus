# Accelerating workloads with NVIDIA GPUs on Red Hat Device Edge

On January 31st, 2023, Red Hat has released Red Hat Device Edge, that provides access to Microshift, which has the simplicity of single-node deployment with the functions and services you need for computing in resource-constrained locations. You can have many deployments on different hosts, creating the specific system image needed for each of your applications. Installing MicroShift on top of your managed RHEL devices in hard-to-service locations also allows for streamlined over-the-air updates.

In this blog we will demonstrate how to enable NVIDIA GPUs on an x86 system running Red Hat Device Edge.

## Pre-requisites

We assume that you have already followed the [Microshift documentation](https://54388--docspreview.netlify.app/microshift/latest/microshift_install/microshift-install-rpm.html) to install it on the Red Hat Enterprise Linux 8.7 machine.

And obviously, you need a machine with an NVIDIA GPU. You can verify this with the following command:

```bash
$ lspci -nnv | grep -i nvidia

example output:
17:00.0 3D controller [0302]: NVIDIA Corporation GA100GL [A30 PCIe] [10de:20b7] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:1532]
```

## Install the NVIDIA GPU driver

This is a fairly simple step, because [NVIDIA provides precompiled driver in RPM repositories that implement the modularity mechanism](https://developer.nvidia.com/blog/streamlining-nvidia-driver-deployment-on-rhel-8-with-modularity-streams/). At this stage, you should have already subscribed your machine and enabled the `rhel-8-for-x86_64-baseos-rpms` and `rhel-8-for-x86_64-appstream-rpms` repositories. We simply need to add the NVIDIA repository.

```bash
$ sudo dnf config-manager --add-repo=https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
```

NVIDIA provides different branches of their drivers, with different lifecycles, which are described in [NVIDIA Datacenter Drivers documentation](https://docs.nvidia.com/datacenter/tesla/drivers/index.html#cuda-drivers). In this blog post, we will use the latest version from the production branch. At the time of writing, it is version R525. The install is done as follows:

```bash
$ sudo dnf module install nvidia-driver:525
```

Now, the driver is installed, be you still need to blacklist the `nouveau` driver which is a community developped in-tree driver from NVIDIA GPUs. It will conflict with the NVIDIA driver and must be disabled. This requires a reboot to take effect.

```bash
$ echo 'blacklist nouveau' | sudo tee /etc/modprobe.d/nouveau-blacklist.conf
$ sudo systemctl reboot
```

After the machine has rebooted, you can verify that the NVIDIA drivers are installed properly

```bash
$ nvidia-smi

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

## Install NVIDIA Container Toolkit

The [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/overview.html) allows users to build and run GPU accelerated containers. The toolkit includes a container runtime library and utilities to automatically configure containers to leverage NVIDIA GPUs. You have to install it to allow the container runtime to transparently configure the NVIDIA GPUs for the pods deployed in Microshift.

NVIDIA container toolkit supports [these distributions](https://nvidia.github.io/nvidia-docker/). Now we install Nvidia Container Toolkit for RHEL 8.6 which matches our version (shown in step 1).
```bash
$ curl -s -L https://nvidia.github.io/libnvidia-container/rhel8.7/libnvidia-container.repo | sudo tee /etc/yum.repos.d/libnvidia-container.repo
$ sudo dnf install nvidia-container-toolkit -y
```

The NVIDIA Container Toolkit requires some SELinux permissions to work properly. These permissions are set in 3 steps.

First, we allow containers to use devices from the host.

```bash
$ sudo setsebool -p container_use_devices on
```

Then, we apply the policy for NVIDIA container toolkit.

```bash
$ curl -sLO https://raw.githubusercontent.com/NVIDIA/dgx-selinux/master/bin/RHEL8/nvidia-container.pp
$ sudo semodule -i nvidia-container.pp
```

It still misses a permission and we can create a policy file.

```bash
$ cat <<EOF > nvidia-container-microshift.te
module nvidia-container-microshift 1.0;

require {
	type xserver_misc_device_t;
	type container_t;
	class chr_file { map read write };
}

#============= container_t ==============
allow container_t xserver_misc_device_t:chr_file map;
EOF
```

And we compile and apply the policy.

```bash
$ checkmodule -m -M -o nvidia-container-microshift.mod nvidia-container-microshift.te
$ semodule_package --outfile nvidia-container-microshift.pp --module nvidia-container-microshift.mod
$ sudo semodule -i nvidia-container-microshift.pp
```

Finally, we restore the file context of the files used by the NVIDIA container toolkit.

```bash
$ nvidia-container-cli -k list | restorecon -v -f -
```


## Install NVIDIA Device Plugin

For Microshift to be able to allocate GPU resource to the pods, you need to deploy the [NVIDIA Device Plugin](https://github.com/NVIDIA/k8s-device-plugin), which is Daemonset that allows you to automatically:

* Expose the number of GPUs on each nodes of your cluster
* Keep track of the health of your GPUs
* Run GPU enabled containers in your Kubernetes cluster.

The deployment is pretty easy, since you only have to add a few manifests and a `kustomize` configuration to the `/etc/microshift/manifests` folder where Microshift looks for manifests to create at start time. This is explained in the [Configuring section of the Microshift documentation](http://docs.openshift.com/microshift/main/microshift_configuring/microshift-using-config-tools.html).

Let's create the folder:

```bash
$ sudo mkdir -p /etc/microshift/manifests
```

To isolate the device plugin from other workloads, we make it run in its own namespace, `nvidia-device-plugin`:

```bash
$ cat <<EOF | sudo tee -a /etc/microshift/manifests/namespace-nvidia-device-plugin.yml
---
apiVersion: v1
kind: Namespace
metadata:
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
  name: nvidia-device-plugin
EOF
```

The device plugin requires to run in `privileged` mode to be able to access the `/var/lib/kubelet/device-plugins` folder on the host. And it also requires the ability to read the node information. So, we create a cluster role that grants these permissions:

```bash
$ cat <<EOF | sudo tee -a /etc/microshift/manifests/role-nvidia-device-plugin.yml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: nvidia-device-plugin
  namespace: nvidia-device-plugin
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - security.openshift.io
  resourceNames:
  - privileged
  resources:
  - securitycontextconstraints
  verbs:
  - use
EOF
```

We create a service account that will be used to run the device plugin pod:

```bash
$ cat <<EOF | sudo tee -a /etc/microshift/manifests/serviceaccount-nvidia-device-plugin.yml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nvidia-device-plugin
  namespace: nvidia-device-plugin
EOF
```

And we associate the service account and the cluster role in a cluster role binding:

```bash
$ cat <<EOF | sudo tee -a /etc/microshift/manifests/rolebinding-nvidia-device-plugin.yml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nvidia-device-plugin
  namespace: nvidia-device-plugin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nvidia-device-plugin
subjects:
  - kind: ServiceAccount
    name: nvidia-device-plugin
    namespace: nvidia-device-plugin
EOF
```

We are now ready to create the DaemonSet specification. You can see that the `securityContext.privileged` field is set to true and that the `serviceAccountName` corresponds to the service account we specified earlier.

```bash
$ cat <<EOF | sudo tee -a /etc/microshift/manifests/daemonset-nvidia-device-plugin.yml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: nvidia-device-plugin
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      priorityClassName: "system-node-critical"
      containers:
        - image: nvcr.io/nvidia/k8s-device-plugin:v0.13.0
          name: nvidia-device-plugin-ctr
          env:
            - name: FAIL_ON_INIT_ERROR
              value: "false"
          securityContext:
            privileged: true
          volumeMounts:
            - name: device-plugin
              mountPath: /var/lib/kubelet/device-plugins
      serviceAccountName: nvidia-device-plugin
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
EOF
```

The resources will not be created automatically just because the files exist. We need to add them to the `kustomize` configuration. This is done through a single `kustomization.yaml` file in the manifests folder that references all the resources we want to create.

```bash
$ cat <<EOF | sudo tee /etc/microshift/manifests/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace-nvidia-device-plugin.yml
  - role-nvidia-device-plugin.yml
  - serviceaccount-nvidia-device-plugin.yml
  - rolebinding-nvidia-device-plugin.yml
  - daemonset-nvidia-device-plugin.yml
EOF
```

At that point, you are ready to restart the `microshift` service to ensure it creates the resources.

```bash
$ sudo systemctl restart microshift
```

Once Microshift is up and running, you can see that the pod is running in `nvidia-device-plugin` namespace.

```bash
$ oc get pod -n nvidia-device-plugin
NAMESPACE                  NAME                                   READY   STATUS              RESTARTS       AGE
nvidia-device-plugin       nvidia-device-plugin-daemonset-jx8s8   1/1     Running             0              1m
```

And you can see in its log that it has registered itself as a device plugin for the `nvidia.com/gpu` resources.

```bash
$ oc logs -n nvidia-device-plugin nvidia-device-plugin-jx8s8
[...]
2022/12/13 04:17:38 Retreiving plugins.
2022/12/13 04:17:38 Detected NVML platform: found NVML library
2022/12/13 04:17:38 Detected non-Tegra platform: /sys/devices/soc0/family file not found
2022/12/13 04:17:38 Starting GRPC server for 'nvidia.com/gpu'
2022/12/13 04:17:38 Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
2022/12/13 04:17:38 Registered device plugin for 'nvidia.com/gpu' with Kubelet
```

You can also verify that the node exposes the `nvidia.com/gpu` resources in its capacity.

```bash
$ oc get node -o json | jq -r '.items[0].status.capacity'

example output:
{
  "cpu": "48",
  "ephemeral-storage": "142063152Ki",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "196686216Ki",
  "nvidia.com/gpu": "2",
  "pods": "250"
}
```

## Run a test GPU accelerate workload

Now that the device plugin is running, you can run workloads that leverage the acceleration.
A simple example the CUDA vectorAdd program. NVIDIA provides it as a container image, so it's easy to use.

We create a `test` namespace.

```bash
$ oc create namespace test
```

And define the pod specification. Note the `spec.containers[0].resources.limits` field where we ask for 1 `nvidia.com/gpu` resource.

```bash
$ cat << EOF > pod-cuda-vector-add.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: test-cuda-vector-add
  namespace: test
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-vector-add
    image: "nvidia/samples:vectoradd-cuda11.2.1-ubi8"
    resources:
      limits:
        nvidia.com/gpu: 1
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      seccompProfile:
        type: "RuntimeDefault"
EOF
```

We can simply create the pod.

```bash
$ oc apply -f pod-cuda-vector-add.yaml
```

And you can see in the pod log that it has found a CUDA device.

```bash
$ oc logs -n test test-cuda-vector-add
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

Well done! You have a machine with Microshift that can run NVIDIA GPU accelerated workloads.

## Run MLPerf Inference and see the performance of real world workloads 
<img width="1432" alt="MLPerf Inference Microshift" src="https://user-images.githubusercontent.com/3208719/215785392-ed796c47-847f-4404-9a25-8febc8699690.png">

