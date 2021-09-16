Refer: 
https://github.com/huyng14/bmwg-container-network/blob/main/roles/sriov-nic-init/tasks/main.yml



# Prequistes
## Create VFs
- Checking supported maximum VFs

```
[root@worker ~]# cat /sys/class/net/enp175s0f0/device/sriov_totalvfs
64
```

- Checking existing VFs

```
[root@worker ~]# cat /sys/class/net/enp175s0f0/device/sriov_numvfs
0
```

- reset SR-IOV Virtual Functions

```
echo 0 > /sys/class/net/enp175s0f0/device/sriov_numvfs
echo 0 > /sys/class/net/enp175s0f1/device/sriov_numvfs
```

- enable SR-IOV Virtual Functions

```
echo 2 > /sys/class/net/enp175s0f0/device/sriov_numvfs
echo 2 > /sys/class/net/enp175s0f1/device/sriov_numvfs
```

```
> [root@worker ~]# ip link
> 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
>     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
> 2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
>     link/ether a4:bf:01:3e:22:e3 brd ff:ff:ff:ff:ff:ff
> 3: eno2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
>     link/ether a4:bf:01:3e:22:e4 brd ff:ff:ff:ff:ff:ff
> 4: enp175s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
>     link/ether 3c:fd:fe:ec:4c:88 brd ff:ff:ff:ff:ff:ff
>     vf 0 MAC 3e:66:f7:2e:68:31, spoof checking on, link-state auto, trust off
>     vf 1 MAC 3a:6b:5b:54:6d:43, spoof checking on, link-state auto, trust off
> 5: enp175s0f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
>     link/ether 3c:fd:fe:ec:4c:89 brd ff:ff:ff:ff:ff:ff
>     vf 0 MAC c6:b6:46:03:69:9e, spoof checking on, link-state auto, trust off
>     vf 1 MAC 62:8e:68:32:4f:a1, spoof checking on, link-state auto, trust off
> 6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
>     link/ether 02:42:c0:c7:2e:c9 brd ff:ff:ff:ff:ff:ff
> 7: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default
>     link/ether aa:2f:ab:c9:96:2b brd ff:ff:ff:ff:ff:ff
```

```
> [root@worker ~]# lspci | grep "Virtual Function"
> af:02.0 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
> af:02.1 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
> af:0a.0 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
> af:0a.1 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
```
- Optionally, make configuration persistent by appending to `rc.local`

## Bring up PF interfaces

```
ip link set enp175s0f0 up
ip link set enp175s0f1 up
```

## Set VFs driver

**Using vfio-pci driver for XL710qda2 NIC**

```
Example output:
[root@worker dpdk-stable-19.11.2]# modprobe vfio-pci
[root@worker dpdk-stable-19.11.2]# lsmod |grep vfio-pci
[root@worker dpdk-stable-19.11.2]# lsmod |grep vf
vfio_pci               41993  0
vfio_iommu_type1       22440  0
vfio                   32657  2 vfio_iommu_type1,vfio_pci
vfat                   17461  1
fat                    65950  1 vfat
irqbypass              13503  2 kvm,vfio_pci
```

- attach VFs driver, Bind VF drivers

maybe using dpdk/usertool/dpdk-bind.py to add driver to VF

```
[root@worker usertools]# ./dpdk-devbind.py -b vfio-pci 0000:af:02.0 0000:af:02.1 0000:af:0a.0 0000:af:0a.1
[root@worker usertools]# ./dpdk-devbind.py -s

Network devices using DPDK-compatible driver
============================================
0000:af:02.0 'Ethernet Virtual Function 700 Series 154c' drv=vfio-pci unused=iavf
0000:af:02.1 'Ethernet Virtual Function 700 Series 154c' drv=vfio-pci unused=iavf
0000:af:0a.0 'Ethernet Virtual Function 700 Series 154c' drv=vfio-pci unused=iavf
0000:af:0a.1 'Ethernet Virtual Function 700 Series 154c' drv=vfio-pci unused=iavf

Network devices using kernel driver
===================================
0000:3d:00.0 'Ethernet Connection X722 for 10GBASE-T 37d2' if=eno1 drv=i40e unused=vfio-pci *Active*
0000:3d:00.1 'Ethernet Connection X722 for 10GBASE-T 37d2' if=eno2 drv=i40e unused=vfio-pci
0000:af:00.0 'Ethernet Controller XL710 for 40GbE QSFP+ 1583' if=enp175s0f0 drv=i40e unused=vfio-pci
0000:af:00.1 'Ethernet Controller XL710 for 40GbE QSFP+ 1583' if=enp175s0f1 drv=i40e unused=vfio-pci
```

## Turn on DMAR|IOMMU

Enable IOMMU on Host
```
> [root@worker usertools]# cat /etc/default/grub
> GRUB_TIMEOUT=5
> GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
> GRUB_DEFAULT=saved
> GRUB_DISABLE_SUBMENU=true
> GRUB_TERMINAL_OUTPUT="console"
> GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet **intel_iommu=on** iommu=pt"
> GRUB_DISABLE_RECOVERY="true"
```
```
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
reboot
```
```
> [root@worker usertools]# dmesg |grep -E "DMAR|IOMMU"
> [    0.000000] ACPI: DMAR 000000006c020000 001D0 (v01 INTEL  S2600WF  00000001 INTL 20091013)
> [    0.000000] DMAR: IOMMU enabled
> [    0.157292] DMAR: Host address width 46
> [    0.157295] DMAR: DRHD base: 0x000000d37fc000 flags: 0x0
> [    0.157303] DMAR: dmar0: reg_base_addr d37fc000 ver 1:0 cap 8d2078c106f0466 ecap f020de
> [    0.157305] DMAR: DRHD base: 0x000000e0ffc000 flags: 0x0
> [    0.157310] DMAR: dmar1: reg_base_addr e0ffc000 ver 1:0 cap 8d2078c106f0466 ecap f020de
> [    0.157312] DMAR: DRHD base: 0x000000ee7fc000 flags: 0x0
> [    0.157318] DMAR: dmar2: reg_base_addr ee7fc000 ver 1:0 cap 8d2078c106f0466 ecap f020de
> [    0.157319] DMAR: DRHD base: 0x000000fbffc000 flags: 0x0
> [    0.157325] DMAR: dmar3: reg_base_addr fbffc000 ver 1:0 cap 8d2078c106f0466 ecap f020de
> [    0.157326] DMAR: DRHD base: 0x000000aaffc000 flags: 0x0
> [    0.157331] DMAR: dmar4: reg_base_addr aaffc000 ver 1:0 cap 8d2078c106f0466 ecap f020de
> [    0.157333] DMAR: DRHD base: 0x000000b87fc000 flags: 0x0
> [    0.157338] DMAR: dmar5: reg_base_addr b87fc000 ver 1:0 cap 8d2078c106f0466 ecap f020de
> [    0.157339] DMAR: DRHD base: 0x000000c5ffc000 flags: 0x0
> [    0.157345] DMAR: dmar6: reg_base_addr c5ffc000 ver 1:0 cap 8d2078c106f0466 ecap f020de
> [    0.157346] DMAR: DRHD base: 0x0000009d7fc000 flags: 0x1
> [    0.157351] DMAR: dmar7: reg_base_addr 9d7fc000 ver 1:0 cap 8d2078c106f0466 ecap f020de
> [    0.157353] DMAR: RMRR base: 0x0000006ba9c000 end: 0x0000006ba9efff
> [    0.157354] DMAR: ATSR flags: 0x0
> [    0.157356] DMAR: ATSR flags: 0x0
> [    0.157359] DMAR-IR: IOAPIC id 12 under DRHD base  0xc5ffc000 IOMMU 6
> [    0.157360] DMAR-IR: IOAPIC id 11 under DRHD base  0xb87fc000 IOMMU 5
> [    0.157362] DMAR-IR: IOAPIC id 10 under DRHD base  0xaaffc000 IOMMU 4
> [    0.157364] DMAR-IR: IOAPIC id 18 under DRHD base  0xfbffc000 IOMMU 3
> [    0.157365] DMAR-IR: IOAPIC id 17 under DRHD base  0xee7fc000 IOMMU 2
> [    0.157367] DMAR-IR: IOAPIC id 16 under DRHD base  0xe0ffc000 IOMMU 1
> [    0.157368] DMAR-IR: IOAPIC id 15 under DRHD base  0xd37fc000 IOMMU 0
> [    0.157370] DMAR-IR: IOAPIC id 8 under DRHD base  0x9d7fc000 IOMMU 7
> [    0.157371] DMAR-IR: IOAPIC id 9 under DRHD base  0x9d7fc000 IOMMU 7
> [    0.157373] DMAR-IR: HPET id 0 under DRHD base 0x9d7fc000
> [    0.157375] DMAR-IR: x2apic is disabled because BIOS sets x2apic opt out bit.
> [    0.157376] DMAR-IR: Use 'intremap=no_x2apic_optout' to override the BIOS setting.
> [    0.160443] DMAR-IR: Enabled IRQ remapping in xapic mode
> [    2.411004] DMAR: dmar6: Using Queued invalidation
> [    2.411012] DMAR: dmar5: Using Queued invalidation
> [    2.411020] DMAR: dmar3: Using Queued invalidation
> [    2.411026] DMAR: dmar2: Using Queued invalidation
> [    2.411035] DMAR: dmar0: Using Queued invalidation
> [    2.411042] DMAR: dmar7: Using Queued invalidation
> [    2.414476] DMAR: Hardware identity mapping for device 0000:00:00.0
> [    2.414479] DMAR: Hardware identity mapping for device 0000:00:04.0
> [    2.414481] DMAR: Hardware identity mapping for device 0000:00:04.1
> [    2.414484] DMAR: Hardware identity mapping for device 0000:00:04.2

> [    2.415233] DMAR: Hardware identity mapping for device 0000:d7:16.0
> [    2.415236] DMAR: Hardware identity mapping for device 0000:d7:16.4
> [    2.415239] DMAR: Hardware identity mapping for device 0000:d7:17.0
> [    2.415240] DMAR: Setting RMRR:
> [    2.415242] DMAR: Ignoring identity map for HW passthrough device 0000:00:14.0 [0x6ba9c000 - 0x6ba9efff]
> [    2.415246] DMAR: Prepare 0-16MiB unity mapping for LPC
> [    2.415247] DMAR: Ignoring identity map for HW passthrough device 0000:00:1f.0 [0x0 - 0xffffff]
> [    2.415258] DMAR: Intel(R) Virtualization Technology for Directed I/O
> [  674.153550] DMAR: 64bit 0000:af:02.0 uses identity mapping
> [  674.158783] DMAR: 64bit 0000:af:02.1 uses identity mapping
> [  678.248078] DMAR: 64bit 0000:af:0a.0 uses identity mapping
> [  678.252061] DMAR: 64bit 0000:af:0a.1 uses identity mapping
```
Refer: https://www.server-world.info/en/note?os=CentOS_7&p=kvm&f=10

# C1.Build SR-IOV CNI

Refer:

https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin#quick-start

```
$ root@master ~/sriov-installer# git clone https://github.com/k8snetworkplumbingwg/sriov-cni.git
$ cd sriov-cni
$ make
$ cp build/sriov /opt/cni/bin
```

**Note: copy build/sriov to all nodes**

# C2.Build and run SR-IOV network device plugin

```
root@master ~/sriov-installer# git clone https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin.git
cd sriov-network-device-plugin
```

# C3. Download or build docker image binary

- On all nodes

```
docker pull nfvpe/sriov-device-plugin
```

On a successful build, a docker image with tag `nfvpe/sriov-device-plugin:latest` will be created. You will need to build this image on each node. Alternatively, you could use a local docker registry to host this image.

# C4.Create a ConfigMap that defines SR-IOV resource pool configuration

Refer: https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin#configurations

```
> [root@worker ~]# lspci -vmmnn |grep -i ethernet
> Class:  Ethernet controller [0200]
> Device: Ethernet Connection X722 for 10GBASE-T [37d2]
> Class:  Ethernet controller [0200]
> Device: Ethernet Connection X722 for 10GBASE-T [37d2]
> Class:  Ethernet controller [0200]
> Device: Ethernet Controller XL710 for 40GbE QSFP+ [1583]
> SDevice:        Ethernet Converged Network Adapter XL710-Q2 [0001]
> Class:  Ethernet controller [0200]
> Device: Ethernet Controller XL710 for 40GbE QSFP+ [1583]
> SDevice:        Ethernet Converged Network Adapter XL710-Q2 [0000]
> Class:  Ethernet controller [0200]
> **Device: Ethernet Virtual Function 700 Series [154c]**
> **Class:  Ethernet controller [0200]**
> Device: Ethernet Virtual Function 700 Series [154c]
> Class:  Ethernet controller [0200]
> Device: Ethernet Virtual Function 700 Series [154c]
> Class:  Ethernet controller [0200]
> Device: Ethernet Virtual Function 700 Series [154c]

> root@master ~/s/s/deployments (master)# cat configMap.yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: sriovdp-config
>   namespace: kube-system
> data:
>   config.json: |
>     {
>         "resourceList": [{
>                 "resourceName": "intel_sriov_netdevice",
>                 "selectors": {
>                     "vendors": ["8086"],
>                     "devices": ["154c", "10ed"],
>                     "drivers": ["i40e"]
>                 }
>             },
>             {
>                 "resourceName": "intel_sriov_dpdk1",
>                 "selectors": {
>                     "vendors": ["8086"],
>                     "devices": ["154c", "10ed"],
>                     "drivers": ["vfio-pci"],
>                     "pfNames": ["enp175s0f0"]
>                 }
>             },
>             {
>                 "resourceName": "intel_sriov_dpdk2",
>                 "selectors": {
>                     "vendors": ["8086"],
>                     "devices": ["154c", "10ed"],
>                     "drivers": ["vfio-pci"],
>                     "pfNames": ["enp175s0f1"]
>                 }
>             }
>         ]
>     }
> root@master ~/s/s/deployments (master)#
```


```
root@master ~/s/s/deployments (master)# kubectl create -f configMap.yaml
```

```
> root@master ~/s/s/deployments (master) [1]# kubectl get configmaps -A
> NAMESPACE         NAME                                 DATA   AGE
> default           kube-root-ca.crt                     1      113d
> kube-node-lease   kube-root-ca.crt                     1      113d
> kube-public       cluster-info                         1      113d
> kube-public       kube-root-ca.crt                     1      113d
> kube-system       calico-config                        4      106d
> kube-system       coredns                              1      113d
> kube-system       extension-apiserver-authentication   6      113d
> kube-system       kube-flannel-cfg                     2      105d
> kube-system       kube-proxy                           2      113d
> kube-system       kube-root-ca.crt                     1      113d
> kube-system       kubeadm-config                       2      113d
> kube-system       kubelet-config-1.20                  1      113d
> kube-system       multus-cni-config                    1      99d
> kube-system       **sriovdp-config**                       1      67s
```


# C5.Deploy SR-IOV network device plugin Daemonset

```
root@master ~/s/s/d/k8s-v1.16 (master)# pwd
/root/sriov-installer/sriov-network-device-plugin/deployments/k8s-v1.16
root@master ~/s/s/d/k8s-v1.16 (master)# kubectl create -f sriovdp-daemonset.yaml
serviceaccount/sriov-device-plugin created
daemonset.apps/kube-sriov-device-plugin-amd64 created
daemonset.apps/kube-sriov-device-plugin-ppc64le created
daemonset.apps/kube-sriov-device-plugin-arm64 created
```

```
> root@master ~/s/s/d/k8s-v1.16 (master)# kubectl get pod -A
> NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
> kube-system   coredns-74ff55c5b-5r8jk                1/1     Running   7          113d
> kube-system   coredns-74ff55c5b-sk5pr                1/1     Running   7          113d
> kube-system   etcd-master                            1/1     Running   11         113d
> kube-system   kube-apiserver-master                  1/1     Running   35         113d
> kube-system   kube-controller-manager-master         1/1     Running   20         113d
> kube-system   kube-flannel-ds-6fpbf                  1/1     Running   15         105d
> kube-system   kube-flannel-ds-f5njk                  1/1     Running   30         105d
> kube-system   kube-multus-ds-amd64-9mjvx             1/1     Running   7          99d
> kube-system   kube-multus-ds-amd64-vc926             1/1     Running   26         99d
> kube-system   kube-proxy-sqf6l                       1/1     Running   7          113d
> kube-system   kube-proxy-xqtj6                       1/1     Running   28         113d
> kube-system   kube-scheduler-master                  1/1     Running   17         113d
> kube-system   **kube-sriov-device-plugin-amd64-8mgjc**   1/1     Running   0          69s
> kube-system   **kube-sriov-device-plugin-amd64-zrbxf**   1/1     Running   0          69s
```


# C6. Deploy SR-IOV network attachment definition

```
[root@master my-sriov]# kubectl create -f netAttach-sriov-dpdk1.yaml
[root@master my-sriov]# kubectl create -f netAttach-sriov-dpdk2.yaml
```

```
[root@master my-sriov]# cat netAttach-sriov-dpdk1.yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-net-1
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/intel_sriov_dpdk1
spec:
  config: '{
  "type": "sriov",
  "cniVersion": "0.3.1",
  "name": "sriov-network-2",
  "vlan": 100,
  "vlanQoS": 1,
  "spoofchk": "off",
  "trust": "on"
}'

[root@master my-sriov]# cat netAttach-sriov-dpdk2.yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-net-2
  annotations:
    k8s.v1.cni.cncf.io/resourceName: intel.com/intel_sriov_dpdk2
spec:
  config: '{
  "type": "sriov",
  "cniVersion": "0.3.1",
  "name": "sriov-network-2",
  "vlan": 200,
  "vlanQoS": 1,
  "spoofchk": "off",
  "trust": "on"
}'
```



## Check if SR-IOV established in worker node

```
[root@master my-sriov]# sudo yum install jq -y
[root@master my-sriov]# kubectl get node worker -o json | jq '.status.allocatable'
{
  "cpu": "80",
  "ephemeral-storage": "676127046561",
  "hugepages-1Gi": "237Gi",
  "intel.com/intel_sriov_dpdk1": "2",
  "intel.com/intel_sriov_dpdk2": "2",
  "memory": "15083532Ki",
  "pods": "110"
}
```

# C7.Deploy SR-IOV pod

```
docker pull huyng14/dpdkapp-v19.08

```

```
root@master ~/s/my-sriov# cat sriov-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sriov-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: sriov-net-1, sriov-net-2
spec:
  containers:
  - name: sriov-pod
    image: huyng14/dpdkapp-v19.08:latest
    imagePullPolicy: IfNotPresent
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/podnetinfo
      name: podnetinfo
      readOnly: false
    - mountPath: /dev/hugepages
      name: hugepage
    resources:
      requests:
        memory: 2Mi
        hugepages-1Gi: 1Gi
        intel.com/intel_sriov_dpdk1: '1'
        intel.com/intel_sriov_dpdk2: '1'
      limits:
        hugepages-1Gi: 1Gi
        intel.com/intel_sriov_dpdk1: '1'
        intel.com/intel_sriov_dpdk2: '1'
    command: ["sleep", "infinity"]
#  nodeSelector:
#    vswitch: ovs
  volumes:
  - name: podnetinfo
    downwardAPI:
      items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
  - name: hugepage
    emptyDir:
      medium: HugePages
```

```
[root@sriov-pod /]# l2fwd -n 4 -l 38-39 --master-lcore 38 --socket-mem=0,1024 -w 0000:af:02.0 -w 0000:af:0a.0 -- -p 0x3 -T 120 --no-mac-updating
```



# Note

- if intel.com/intel_sriov_dpdk1": "0" as below

```
Example output:
root@master ~/s/my-sriov# kubectl get node worker -o json | jq '.status.allocatable'
{
  "cpu": "80",
  "ephemeral-storage": "676127046561",
  "hugepages-1Gi": "237Gi",
  "intel.com/intel_sriov_dpdk1": "0",
  "intel.com/intel_sriov_dpdk2": "0",
  "memory": "15083528Ki",
  "pods": "110"
}
```

```
root@master ~/s/my-sriov# kubectl delete -f sriovdp-daemonset.yaml
serviceaccount "sriov-device-plugin" deleted
daemonset.apps "kube-sriov-device-plugin-amd64" deleted
daemonset.apps "kube-sriov-device-plugin-ppc64le" deleted
daemonset.apps "kube-sriov-device-plugin-arm64" deleted

root@master ~/s/my-sriov# kubectl create -f sriovdp-daemonset.yaml
serviceaccount/sriov-device-plugin created
daemonset.apps/kube-sriov-device-plugin-amd64 created
daemonset.apps/kube-sriov-device-plugin-ppc64le created
daemonset.apps/kube-sriov-device-plugin-arm64 created
```

