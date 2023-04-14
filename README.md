# Using VFs in OpenShift with SRIOV and DPDK

It is possible to use Virtual Functions (vfs) passed through to an OpenShift worker node and then manage them as a VF within OpenShift. This is NOT a supported configuration, but can be completed by following the process below. Note that the instructions below are designed for setting up a small test environment and will need to be modified to meet your specific needs.

## Prerequisites  

* **butane** - https://github.com/coreos/butane/releases

## Setup

The following was tested with the following:

* HP DL360p Gen8
* Intel X520-DA1 10G network card (Intel 82599ES Chip-set)
* Fedora 36 as host machine
* OpenShift 4.11 - Single Node OpenShift Cluster

## Configuration

### Enable VFs and SRIOV on HOST hardware

The following steps must be taken to configure the HOST hardware to make SRIOV VFs available to a Virtual Machine. The following steps were used on a Fedora 36 Workstation install. The instructions also assume the use of Intel 82599 card(s) for creating SRIOV VFs. 

Create the following files:

/etc/modules.d/ixgbe_sriov.conf
```conf
# enable sriov vfs on intel card
options ixgbe max_vfs=8
```

/etc/modules.d/iommu.conf
```conf
# enable vfios iommu. 
# ensure that "unsafe interrupts" is enabled
options vfio_iommu_type1 allow_unsafe_interrupts=1
```

Make virtual functions permanent after reboot
/etc/udev/rules.d/ixgbe.rules
```
ACTION=="add", SUBSYSTEM=="net", ENV{ID_NET_DRIVER}=="ixgbe",
ATTR{device/sriov_numvfs}="8"
```

/etc/modules-load.d/vfio.conf
```
vfio 
vfio_pci 
vfio_iommu_type1
```

Ensure IOMMU is enabled at boot time 

```
grubby --args="intel_iommu=on iommu=pt rd.driver.pre=vfio-pci --update-kernel=ALL"
```

Reboot your host node to apply the configuration above. Once the host has completed the reboot run `lspci` to validate that virtual functions were created.

```
$ sudo lspci
...
04:00.0 Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection (rev 01)
04:10.0 Ethernet controller: Intel Corporation 82599 Ethernet Controller Virtual Function (rev 01)
04:10.2 Ethernet controller: Intel Corporation 82599 Ethernet Controller Virtual Function (rev 01)
04:10.4 Ethernet controller: Intel Corporation 82599 Ethernet Controller Virtual Function (rev 01)
04:10.6 Ethernet controller: Intel Corporation 82599 Ethernet Controller Virtual Function (rev 01)
04:11.0 Ethernet controller: Intel Corporation 82599 Ethernet Controller Virtual Function (rev 01)
04:11.2 Ethernet controller: Intel Corporation 82599 Ethernet Controller Virtual Function (rev 01)
04:11.4 Ethernet controller: Intel Corporation 82599 Ethernet Controller Virtual Function (rev 01)
04:11.6 Ethernet controller: Intel Corporation 82599 Ethernet Controller Virtual Function (rev 01)
```

The output above will be used to configure your virtual machine(s) going forward


## VM Configuration

Instructions below are based on using KVM as your virtual machine hypervisor. You will need to modify the settings below based on your specific needs.

You will need to edit your vm definition and add an IOMMU to the devices section:

```xml
<iommu model='intel'>
    <driver intremap='on' caching_mode='on'/>
</iommu>
```

Add ioapci driver to the Features section
```
<ioapic driver='qemu'/>
```

You will also need to add a memtune section to the "domain" section of the file:

```xml
<memtune>
    <hard_limit unit='KiB'>104857600</hard_limit>
</memtune>

NOW we can add the virtual functions to our vm. In the devices section add the following:

```
<hostdev mode='subsystem' type='pci' managed='yes'>
  <driver name='vfio'/>
  <source>
    <address domain='0x0000' bus='0x4' slot='0x10' function='0x0'/>
  </source>
  <alias name='hostdev0'/>
  <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
</hostdev>
```

Update source address to match the device you are attempting to pass in.

## Deploying OpenShift

This document will not cover the install of OpenShift. Suggest that you use the [Assisted Installer](https://docs.openshift.com/container-platform/4.12/installing/installing_on_prem_assisted/installing-on-prem-assisted.html) or the [Agent Installer](https://docs.openshift.com/container-platform/4.12/installing/installing_with_agent_based_installer/preparing-to-install-with-agent-based-installer.html) process for deploying your new cluster.

## Configuring OpenShift

### Update ulimits for containers

Create a machineConfiguration that updates the ulimit for all containers

```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  annotations:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 02-worker-container-runtime
spec:
  config:
    ignition:
      version: 3.1.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,W2NyaW8ucnVudGltZV0KZGVmYXVsdF91bGltaXRzID0gWwoibnByb2M9MTYzNDg6LTEiLAoic3RhY2s9MTYwMDAwMDotMSIKXQo=
        mode: 420
        overwrite: true
        path: /etc/crio/crio.conf.d/10-custom
```

This creates a file called /etc/crio/crio.conf.d/10-custom that contains the following:

```conf
[crio.runtime]
default_ulimits = [
"nproc=16348:-1",
"stack=1600000:-1"
]
```

### Configure Machine Config

We will use a machineConfig profile to get 

* hugepages
* enable iommu
* disable ixgbe driver


```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 100-master-blacklist
spec:
  config:
    ignition:
      version: 3.2.0
  kernelArguments:
      - modprobe.blacklist=ixgbevf
      - intel_iommu=on
      - hugepages=1024
```

### configure vfio

If you are not using a Intel82599 card, you will need to update the vfiopci.bu file with the proper PCI device ID and then generate a new vfio=worker-mc.yaml file

```
$ butane vfiopci.bu > vfio-worker-mc.yaml
```

Apply the vfio machineConfig

```
$ oc create -f vfio-worker-mc.yaml
```

### Deploy SRIOV Device Plugin

Update configmap file to reflect the proper device VF that is being passed in if the template file does not contain the right entries. This can be determined by running

```shell
$ oc debug node <>
# chroot /host
# lspci -nn
0:00.0 Host bridge [0600]: Intel Corporation 82G33/G31/P35/P31 Express DRAM Controller [8086:29c0]
00:01.0 VGA compatible controller [0300]: Red Hat, Inc. Virtio GPU [1af4:1050] (rev 01)
...
08:00.0 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
09:00.0 Ethernet controller [0200]: Intel Corporation 82599 Ethernet Controller Virtual Function [8086:10ed] (rev 01)
```


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: sriovdp
data:
  config.json: |
    {
        "resourceList": [
            {
                "resourceName": "intel_sriov_dpdk",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed", "1889"],
                    "drivers": ["vfio-pci"]
                }
            }
        ]
    }
```

```shell
oc new-project sriovdp
oc create sa sriov-device-plugin-sa
oc adm policy add-scc-to-user privileged system:serviceaccount:sriovdp:sriov-device-plugin-sa
oc create -f templates/sriov-device-plugin-config.yml
oc create -f templates/sriov-device-plugin-daemonset.yml
```

## Test deployment

To test that this is working we will use a pod with `testpmd` installed

```shell
$ oc create template/dpdk-pod.yml
$ oc rsh po/testpmd
# testpmd
EAL: Detected CPU lcores: 8
EAL: Detected NUMA nodes: 1
EAL: Detected shared linkage of DPDK
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'PA'
EAL: No available 1048576 kB hugepages reported
EAL: VFIO support initialized
EAL: Probe PCI driver: net_virtio (1af4:1041) device: 0000:01:00.0 (socket 0)
eth_virtio_pci_init(): Failed to init PCI device
EAL: Requested device 0000:01:00.0 cannot be used
EAL: Using IOMMU type 1 (Type 1)
EAL: Probe PCI driver: net_ixgbe_vf (8086:10ed) device: 0000:08:00.0 (socket 0)
EAL: Probe PCI driver: net_ixgbe_vf (8086:10ed) device: 0000:09:00.0 (socket 0)
TELEMETRY: No legacy callbacks, legacy socket not created
testpmd: create a new mbuf pool <mb_pool_0>: n=203456, size=2176, socket=0
testpmd: preferred mempool ops selected: ring_mp_mc
Configuring Port 0 (socket 0)
Port 0: 02:09:C0:B2:EC:44
Configuring Port 1 (socket 0)
Port 1: 02:09:C0:4F:24:DB
Checking link statuses...
Done
No commandline core given, start packet forwarding
io packet forwarding - ports=2 - cores=1 - streams=2 - NUMA support enabled, MP allocation mode: native
Logical Core 1 (socket 0) forwards packets on 2 streams:
  RX P=0/Q=0 (socket 0) -> TX P=1/Q=0 (socket 0) peer=02:00:00:00:00:01
  RX P=1/Q=0 (socket 0) -> TX P=0/Q=0 (socket 0) peer=02:00:00:00:00:00

  io packet forwarding packets/burst=32
  nb forwarding cores=1 - nb forwarding ports=2
  port 0: RX queue number: 1 Tx queue number: 1
    Rx offloads=0x0 Tx offloads=0x0
    RX queue: 0
      RX desc=512 - RX free threshold=32
      RX threshold registers: pthresh=0 hthresh=0  wthresh=0
      RX Offloads=0x0
    TX queue: 0
      TX desc=512 - TX free threshold=32
      TX threshold registers: pthresh=32 hthresh=0  wthresh=0
      TX offloads=0x0 - TX RS bit threshold=32
  port 1: RX queue number: 1 Tx queue number: 1
    Rx offloads=0x0 Tx offloads=0x0
    RX queue: 0
      RX desc=512 - RX free threshold=32
      RX threshold registers: pthresh=0 hthresh=0  wthresh=0
      RX Offloads=0x0
    TX queue: 0
      TX desc=512 - TX free threshold=32
      TX threshold registers: pthresh=32 hthresh=0  wthresh=0
      TX offloads=0x0 - TX RS bit threshold=32
```