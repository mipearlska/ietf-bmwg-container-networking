Refer: 
https://github.com/huyng14/bmwg-container-network/blob/main/roles/sriov-nic-init/tasks/main.yml

# Prequisites
## Docker, Kubernetes cluster v1.23.5, Multus CNI, UserspaceCNI, SRIOV plugin (setup below)
## Userspace and SRIOV CNI bin must be copied at /opt/cni/bin in all nodes
## Setup iommu (guide in 00_PREQUISITES)
## High version T-Rex might not compatible with SRIOV VF, cause traffic to be dropped (our test failed with v2.92). Consider to use v2.73 or lower

# Benchmarking Flow
1. (worker) Setup SRIOV-VF
2. (master) Apply SRIOV plugin configmap, daemonset, network attachment definition
3. (master) Deploy pod 
4. (master) Kubectl exec into pod then run dpdk-l2fwd app
5. (t-rex) Config traffic profile then send traffic from t-rex
6. (t-rex) Run benchmarking NDR app from t-rex


# 1. Setup SR-IOV Virutal Functions at Worker DUT node
### Create VFs
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
echo 1 > /sys/class/net/enp175s0f0/device/sriov_numvfs
echo 1 > /sys/class/net/enp175s0f1/device/sriov_numvfs
```

```
> [root@worker ~]# ip link
4: enp175s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 3c:fd:fe:ec:4c:88 brd ff:ff:ff:ff:ff:ff
    vf 0     link/ether ba:1d:1d:ae:8a:04 brd ff:ff:ff:ff:ff:ff, spoof checking off, link-state auto, trust off
5: enp175s0f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 3c:fd:fe:ec:4c:89 brd ff:ff:ff:ff:ff:ff
    vf 0     link/ether fa:ca:84:fe:8b:a4 brd ff:ff:ff:ff:ff:ff, spoof checking off, link-state auto, trust off

```

```
> [root@worker ~]# lspci | grep "Virtual Function"
af:02.0 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
af:0a.0 Ethernet controller: Intel Corporation Ethernet Virtual Function 700 Series (rev 02)
```
- Optionally, make configuration persistent by appending to `rc.local`

- Bring up PF interfaces

```
ip link set enp175s0f0 up
ip link set enp175s0f1 up
```

- Set VFs driver

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

use dpdk/usertool/dpdk-bind.py to add driver to VF

```
[root@worker usertools]# ./dpdk-devbind.py -b vfio-pci 0000:af:02.0 0000:af:0a.0
[root@worker usertools]# ./dpdk-devbind.py -s

Network devices using DPDK-compatible driver
============================================
0000:af:02.0 'Ethernet Virtual Function 700 Series 154c' drv=vfio-pci unused=iavf
0000:af:0a.0 'Ethernet Virtual Function 700 Series 154c' drv=vfio-pci unused=iavf

Network devices using kernel driver
===================================
0000:3d:00.0 'Ethernet Connection X722 for 10GBASE-T 37d2' if=eno1 drv=i40e unused=vfio-pci *Active*
0000:3d:00.1 'Ethernet Connection X722 for 10GBASE-T 37d2' if=eno2 drv=i40e unused=vfio-pci 
0000:af:00.0 'Ethernet Controller XL710 for 40GbE QSFP+ 1583' if=enp175s0f0 drv=i40e unused=vfio-pci 
0000:af:00.1 'Ethernet Controller XL710 for 40GbE QSFP+ 1583' if=enp175s0f1 drv=i40e unused=vfio-pci 
```

# 2. Setup SRIOV plugin

Refer:
https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin#quick-start

### 2.1 Build SRIOV CNI bin or get directly from this repo 1_cnibin folder
```
$ root@master ~/sriov-installer# git clone https://github.com/k8snetworkplumbingwg/sriov-cni.git
$ cd sriov-cni
$ make
$ cp build/sriov /opt/cni/bin
```
Copy the sriov cni bin to all nodes

### 2.2 Get SRIOV device plugin

- Clone the plugin repo
```
git clone https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin.git
cd sriov-network-device-plugin
```
- Pull the plugin daemonset required image to all nodes
```
 docker pull ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin:latest
```

### 2.3 Modify the SR-IOV resource pool ConfigMap to fit with the DUT system then Apply the ConfigMap

- Check DUT system NIC info
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
```
- Modify config-map.yaml based on the NIC interface info above
```
cd sriov-network-device-plugin/deployments
nano config-map.yaml
```
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
        "resourceList": [{
                "resourceName": "intel_sriov_netdevice",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed"],
                    "drivers": ["i40e"]
                }
            },
            {
                "resourceName": "intel_sriov_dpdk1",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed"],
                    "drivers": ["vfio-pci"],
                    "pfNames": ["enp175s0f0"]
                }
            },
            {
                "resourceName": "intel_sriov_dpdk2",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed"],
                    "drivers": ["vfio-pci"],
                    "pfNames": ["enp175s0f1"]
                }
            }
        ]
    }
```
- Apply the configmap
```
kubectl create -f config-map.yaml
```
- Verify
```
kubectl get configmaps -A
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

### 2.4.Apply the  SR-IOV network device plugin Daemonset

```
cd sriov-network-device-plugin/deployments
kubectl create -f sriovdp-daemonset.yaml
```
```
serviceaccount/sriov-device-plugin created
daemonset.apps/kube-sriov-device-plugin-amd64 created
daemonset.apps/kube-sriov-device-plugin-ppc64le created
daemonset.apps/kube-sriov-device-plugin-arm64 created
```
- Verify
```
kubectl get pod -A

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


### 2.5. Deploy SR-IOV network attachment definition

```
kubectl create -f netAttach-sriov-dpdk1.yaml
kubectl create -f netAttach-sriov-dpdk2.yaml
```

```
cat netAttach-sriov-dpdk1.yaml


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

cat netAttach-sriov-dpdk2.yaml


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
- Verify if SR-IOV established in worker node

```
sudo yum install jq -y
kubectl get node worker41 -o json | jq '.status.allocatable'


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

### 2.6 Deploy SR-IOV l2fwd pod
- App image
```
docker pull mipearlska/dpdk-app-ubuntu
```
- Annotations: 2 SRIOV Network Attachement Definitions name above
- Define the two SRIOV resource (same as in Network Attachment Definition) in the "requests" field
```
cat pod-sriov.yaml
```
```
apiVersion: v1
kind: Pod
metadata:
  name: sriov-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: sriov-net-1, sriov-net-2
spec:
  containers:
  - name: pod-sriov
    image: mipearlska/dpdk-app-ubuntu
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

### 2.7 Run the l2fwd inside pod
** Consider to change numa core (-l 38-39) if running fail
- a: 2 PCI of the SRIOV VFs at worker nodes
- Other parameters: Please refer to https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html
```
kubectl exec -it pod-sriov /bin/bash
./dpdk-l2fwd -n 4 -l 38-39 --socket-mem=0,1024 -a 0000:af:02.0 -a 0000:af:0a.0 -- -p 0x3 -T 30 --no-mac-updating
```

# 3. T-Rex Traffic Generator 
### High version of T-Rex might cause traffic dropped at SRIOV VF, our test failed with v2.92
### This testbed use T-Rex v2.73
### 3.1. Config t-rex (packet src,destination IP/MAC)
- Please change MAC addresses to the corresponding MAC addresses of the VFs
```bash
- nano /etc/trex_cfg.yaml
```
```bash
  port_info:
      - dest_mac: BA:1D:1D:AE:8A:04  # Check from MAC OF port 0 pod-sriov OR attached vf info from "ip link" command
        src_mac:  FA:CA:84:FE:8B:A4  # Check from MAC OF port 1 pod-sriov OR attached vf info from "ip link" command
      - dest_mac: FA:CA:84:FE:8B:A4  # SRIOV scenario
        src_mac:  BA:1D:1D:AE:8A:04
```
### 3.2. Start T-gen (1 tab console)
```bash
cd v2.73
./t-rex-64 -i -c 10
```

### 3.3. Generate traffic (1 tab console)
```bash
cd v2.73
./trex-console
tui
```

### Generate test traffic
```bash
start -f stl/bench.py -p 0 -m 100% --force -t size=1518
start -f stl/bench.py -p 1 -m 100% --force -t size=1518
start -f stl/bench.py -m 100% --force -t size=1518
```
(p = from port 0 or 1 of tgen, no -p flag = generate from both ports, -size = packet size, -m = %max line rate of NIC card)

### 3.4. Benchmark application run (No Drop Rate Benchmarking) - (Manual: https://trex-tgn.cisco.com/trex/doc/trex_ndr_bench_doc.html)
```bash
./ndr --stl --port 0 1 -v --profile stl/bench.py --prof-tun size=1518 --opt-bin-search
```



# Other Notes

- if intel.com/intel_sriov_dpdk1": "0" as below
- Delete and re-create sriovdp-daemonset

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

### Clean up, revert environment After Finishing Benchmark
Master
```
root@master42:~/bmwg/sriov# kubectl delete -f pod-sriov.yaml

root@master42:~/bmwg/sriov/sriov-network-device-plugin/deployments# kubectl delete -f sriovdp-daemonset.yaml
root@master42:~/bmwg/sriov/sriov-network-device-plugin/deployments# kubectl delete -f config-map.yaml

root@master42:~/bmwg/sriov# kubectl delete -f net-attach-sriov-dpdk1.yaml
root@master42:~/bmwg/sriov# kubectl delete -f net-attach-sriov-dpdk2.yaml
```

Worker
```
echo 0 > /sys/class/net/enp175s0f0/device/sriov_numvfs
echo 0 > /sys/class/net/enp175s0f1/device/sriov_numvfs
```


