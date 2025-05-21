Derived from the nvidia and Red Hat docs here:

https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/openshift-virtualization.html#creating-a-clusterpolicy-for-the-gpu-operator-using-the-openshift-container-platform-cli

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/virtualization/virtual-machines#virt-adding-kernel-arguments-enable-IOMMU_virt-configuring-pci-passthrough


NOTE: The most common issue with PCI passthrough is that the nvidia kernel module is loaded on a host which claims the devices instead of vfio claiming the devices.
      A GPU device at the OS level cannot be managed by both the nvidia kernel module and the vfio module at the same time.

- Deploy the GPU operator
- Deploy the OpenShift Virt Operator
- Deploy a cluster policy for the GPU Operator
- Create a HyperConverged instance for the OpenShift Virt Operator


- Enable IOMMU in the kernel
```console
oc create -f 100-worker-kernel-arg-iommu.yaml
```
- Wait for the machine config pool to finish updating the cluster nodes
```console
oc get machineconfigpool
```

- Label Nodes as container, vm-passthrough, or vm-vgpu

```console
oc label node <node-name> --overwrite nvidia.com/gpu.workload.config=vm-passthrough
```

- Run lspci to obtain the vendor-ID and device-ID for the PCI devices to be passed through
Note: This example assumes all the GPU devices are the same across nodes. If there are different GPU devices in the same OCP machine config pool then a custom Machine Config with a script must be created to load vfio with the correct device IDs for each node

```console
oc debug node/my-node -- chroot /host lspci -nnv | grep -i nvidia
```

- Create a butane config that includes the device ID(s) from the previous lspci command and convert to a machine config yaml
```console
butane 100-worker-vfiopci.bu -o 100-worker-vfiopci.yaml
```

- Apply the vfio machine config
```console
oc apply -f 100-worker-vfiopci.yaml
```

- Wait for the machine config pool to finish updating the cluster nodes
```console
oc get machineconfigpool
```

- Update the hyper converged object to specify the pass through devices
```console
oc edit hyperconverged kubevirt-hyperconverged -n openshift-cnv
```

```
apiVersion: hco.kubevirt.io/v1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:
  permittedHostDevices: 1
    pciHostDevices: 2
    - pciDeviceSelector: "10DE:1DB6" 3
      resourceName: "nvidia.com/GV100GL_Tesla_V100" 4
    - pciDeviceSelector: "10DE:1EB8"
      resourceName: "nvidia.com/TU104GL_Tesla_T4"
    - pciDeviceSelector: "8086:6F54"
      resourceName: "intel.com/qat"
      externalResourceProvider: true 5
# ...
```

- Check the nodes to see they now have passthrough device capacity with the updates to the hyper converged CR above
```console
oc describe node <node_name>
```

```
Capacity:
  cpu:                            64
  devices.kubevirt.io/kvm:        110
  devices.kubevirt.io/tun:        110
  devices.kubevirt.io/vhost-net:  110
  ephemeral-storage:              915128Mi
  hugepages-1Gi:                  0
  hugepages-2Mi:                  0
  memory:                         131395264Ki
  nvidia.com/GV100GL_Tesla_V100   1
  nvidia.com/TU104GL_Tesla_T4     1
  intel.com/qat:                  1
  pods:                           250
Allocatable:
  cpu:                            63500m
  devices.kubevirt.io/kvm:        110
  devices.kubevirt.io/tun:        110
  devices.kubevirt.io/vhost-net:  110
  ephemeral-storage:              863623130526
  hugepages-1Gi:                  0
  hugepages-2Mi:                  0
  memory:                         130244288Ki
  nvidia.com/GV100GL_Tesla_V100   1
  nvidia.com/TU104GL_Tesla_T4     1
  intel.com/qat:                  1
  pods:                           250
```