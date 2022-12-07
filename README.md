# Using VFs in OpenShift with SRIOV and DPDK

## Prereques

* **butane** - https://github.com/coreos/butane/releases

## Setup

HP DL360PG10
Intel ixgbe card
Fedora 36
OpenShift 4.11

## Configuration

### Enable VFs and SRIOV 

Create two files:

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

Reboot your host node and then run 

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
      - hugepages=10
```

configure vfio

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
