# Prerequisites
### OS: Ubuntu 22.04 (OS kernel 5.15)
### Install docker, kubernetes v1.23.5, Multus CNI, Userspace CNI 
### Install DPDK 22.11.1 (use meson ninja)
### Install OVS-DPDK 3.1.90 (current latest)
### Userspace CNI bin must be copied to all nodes (/opt/cni/bin) - Should build from Ubuntu 18.04 machine then copy to 22.04 DUT

```bash
sudo apt-get install -qy kubelet=1.23.5-00 kubectl=1.23.5-00 kubeadm=1.23.5-00
```

```bash
install libtool (apt-get install libtool)
install autoconf (apt-get install autoconf)
git clone https://github.com/openvswitch/ovs.git
cd ovs
./boot.sh
./configure --with-dpdk=static
make -j $(nproc)
sudo make install
```

# Benchmarking Flow
0. (t-rex) Config t-rex traffic profile (/etc/trex_cfg.yaml) as below then start t-rex. Not send any traffic yet
1. (master) Apply network attachment definition (check kubectl get net-attach-def)
2. (worker) Start OVS and config OVS flow
3. (master) Deploy pod 
4. (master) Kubectl exec into pod then run dpdk-l2fwd app
5. (t-rex) Send traffic from t-rex
6. (t-rex) Run benchmarking NDR app from t-rex

# 1. Set up packet flow through OVS-DPDK

- attach/bind DPDK driver to benchmarking NIC interface

use dpdk/usertool/dpdk-bind.py to add driver to VF

```
[root@worker usertools]# ./dpdk-devbind.py -b vfio-pci 0000:af:00.0 0000:af:00.1
[root@worker usertools]# ./dpdk-devbind.py -s

Network devices using DPDK-compatible driver
============================================
0000:af:00.0 'Ethernet Controller XL710 for 40GbE QSFP+ 1583' drv=vfio-pci unused=i40e
0000:af:00.1 'Ethernet Controller XL710 for 40GbE QSFP+ 1583' drv=vfio-pci unused=i40e

Network devices using kernel driver
===================================
0000:3d:00.0 'Ethernet Connection X722 for 10GBASE-T 37d2' if=eno1 drv=i40e unused=vfio-pci *Active*
0000:3d:00.1 'Ethernet Connection X722 for 10GBASE-T 37d2' if=eno2 drv=i40e unused=vfio-pci 
```

dpdk-handling core2(lcore-mask), 8 dpdk pmd core 0-8(pmd-cpu-mask), afxdp pmd core 4,6 (pmd-rxq-affinity: queue 0 - core 4,6)
create ovs-br0 bridge in OVS vswitch
```bash
/usr/local/share/openvswitch/scripts/ovs-ctl --no-ovs-vswitchd start
/usr/local/bin/ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="4096,0"
/usr/local/bin/ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-lcore-mask=0x2
/usr/local/bin/ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
/usr/local/bin/ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0xff
/usr/local/share/openvswitch/scripts/ovs-ctl --no-ovsdb-server --db-sock="/usr/local/var/run/openvswitch/db.sock" start
/usr/local/bin/ovs-vsctl del-br ovs-br0
/usr/local/bin/ovs-vsctl --may-exist add-br ovs-br0 -- set bridge ovs-br0 datapath_type=netdev
```

add OVS port (2 NIC interface ports, 2 vhostuser ports)
```bash
ovs-vsctl add-port ovs-br0 gigaf0/0 -- set Interface gigaf0/0 type=dpdk options:dpdk-devargs=0000:af:00.0
ovs-vsctl add-port ovs-br0 gigaf0/1 -- set Interface gigaf0/1 type=dpdk options:dpdk-devargs=0000:af:00.1
ovs-vsctl add-port ovs-br0 dpdkvhostuser0 -- set Interface dpdkvhostuser0 type=dpdkvhostuser
ovs-vsctl add-port ovs-br0 dpdkvhostuser1 -- set Interface dpdkvhostuser1 type=dpdkvhostuser
```

check ovs-bro bridge status
```bash
ovs-vsctl show
ovs-ofctl show ovs-br0
```
```
Example output:
OFPT_FEATURES_REPLY (xid=0x2): dpid:00003cfdfeec4c88
n_tables:254, n_buffers:0
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 1(gigaf0/0): addr:3c:fd:fe:ec:4c:88
     config:     0
     state:      0
     current:    AUTO_NEG
     speed: 0 Mbps now, 0 Mbps max
 2(gigaf0/1): addr:3c:fd:fe:ec:4c:89
     config:     0
     state:      0
     current:    AUTO_NEG
     speed: 0 Mbps now, 0 Mbps max
 3(dpdkvhostuser0): addr:00:00:00:00:00:00
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 4(dpdkvhostuser1): addr:00:00:00:00:00:00
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
```

add traffic routing flows between each NIC interface and vhostuser ports (get port number from show ovs-br0 command above)
```bash
ovs-ofctl --timeout 10 -O Openflow13 add-flow ovs-br0 in_port=1,idle_timeout=0,action=output:3
ovs-ofctl --timeout 10 -O Openflow13 add-flow ovs-br0 in_port=3,idle_timeout=0,action=output:1
ovs-ofctl --timeout 10 -O Openflow13 add-flow ovs-br0 in_port=2,idle_timeout=0,action=output:4
ovs-ofctl --timeout 10 -O Openflow13 add-flow ovs-br0 in_port=4,idle_timeout=0,action=output:2
ovs-ofctl dump-flows ovs-br0
```
```
Example output:
[root@worker dpdk-stable-19.11.2]# ovs-ofctl dump-flows ovs-br0
 cookie=0x0, duration=17466.956s, table=0, n_packets=134504920, n_bytes=203640448880, in_port="gigaf0/0" actions=output:dpdkvhostuser0
 cookie=0x0, duration=17466.918s, table=0, n_packets=2, n_bytes=128, in_port=dpdkvhostuser0 actions=output:"gigaf0/0"
 cookie=0x0, duration=17439.809s, table=0, n_packets=0, n_bytes=0, in_port="gigaf0/1" actions=output:dpdkvhostuser1
 cookie=0x0, duration=17434.264s, table=0, n_packets=134504779, n_bytes=203640232506, in_port=dpdkvhostuser1 actions=output:"gigaf0/1"
```

# 2. L2FWD application

### Userspace CNI Network Attachment Definition
- Get the corresponding OVS bridgeName from above (ovs-br0 in this example)
```
[root@master ovs-dpdk]# cat userspace-ovs-CRD.yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: userspace-ovs-net
spec:
  config: '{
        "cniVersion": "0.3.1",
        "type": "userspace",
        "name": "userspace-ovs-net-1",
        "kubeconfig": "/etc/cni/net.d/multus.d/multus.kubeconfig",
        "logFile": "/var/log/userspace-ovs-net.log",
        "logLevel": "debug",
        "host": {
                "engine": "ovs-dpdk",
                "iftype": "vhostuser",
                "netType": "bridge",
                "vhost": {
                        "mode": "server"
                },
                "bridge": {
                        "bridgeName": "ovs-br0"
                }
        },
        "container": {
                "engine": "ovs-dpdk",
                "iftype": "vhostuser",
                "netType": "interface",
                "vhost": {
                        "mode": "client"
                }
        }
    }'
```



### POD yaml (L2fwd or testpmd)
Image: (dpdk app ver 22.11, ubuntu 22.04)
```
docker pull mipearlska/dpdk-app-ubuntu
```
- Annotations: 2 Userspace Network Attachment Definition's name
- Required Volume to mount: podinfo (default), hugepages (get from worker node), OVS dpdkvhostuser directory (get from worker node)
```
root@master ~/t/u/ovs-dpdk# cat ovs-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ovs-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: userspace-ovs-net, userspace-ovs-net
spec:
  containers:
  - name: multi-vhost
    image: mipearlska/dpdk-app-ubuntu
    imagePullPolicy: IfNotPresent
    terminationMessagePath: "/tmp/ovs-pod/"
#    command: [ "/bin/bash", "-c", "--" ]
#    args: [ "while true; do sleep 30; done;" ]
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/podnetinfo
      name: podinfo
      readOnly: false
    - mountPath: /var/run/openvswitch/
      name: shared-dir
    - mountPath: /dev/hugepages
      name: hugepage
    resources:
      requests:
        memory: 2Gi
        hugepages-1Gi: 1Gi
      limits:
        hugepages-1Gi: 1Gi
    command: ["sleep", "infinity"]
#  nodeSelector:
#    vswitch: ovs
  volumes:
  - name: podinfo
    downwardAPI:
      items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
  - name: shared-dir
    hostPath:
      path: /usr/local/var/run/openvswitch/
  - name: hugepage
    emptyDir:
      medium: HugePages
```

### Running l2fwd application
- vdev: 2 dpdkvhostuser port at OVS vswitch
- l: CPU NUMA cores (check available NUMA from worker by command "numactl -H"/"lspcu")
- n: Number of memory channels
- T: refreshing forwarding status frequency
- Others: please refer from https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html
```bash
cd $DPDK_DIRECTORY/build/examples

./dpdk-l2fwd -n 4 -l 15-18 --single-file-segments --vdev=virtio_user0,path=/var/run/openvswitch/dpdkvhostuser0 --vdev=virtio_user1,path=/var/run/openvswitch/dpdkvhostuser1 --no-pci -- -p 0x3 -T 10 --no-mac-updating

./dpdk-l2fwd -n 4 -l 15-18 --single-file-segments --vdev=virtio_user0,path=/usr/local/var/run/openvswitch/dpdkvhostuser0 --vdev=virtio_user1,path=/usr/local/var/run/openvswitch/dpdkvhostuser1 --no-pci -- -p 0x3 -T 10 --no-mac-updating

l2fwd -n 4 -l 15-18 --single-file-segments --vdev=virtio_user0,path=/var/run/openvswitch/dpdkvhostuser0 --vdev=virtio_user1,path=/var/run/openvswitch/dpdkvhostuser1 --no-pci -- -p 0x3 -T 10 --no-mac-updating
```


--------------------------------------------
# Clean OVS, Check traffic status at interface, OVS PMD

### delete ovs-bridge, shutdown ovs
```bash
ovs-vsctl del-br ovs-br0
/usr/local/share/openvswitch/scripts/ovs-ctl --no-ovsdb-server --db-sock="/usr/local/var/run/openvswitch/db.sock" stop
/usr/local/share/openvswitch/scripts/ovs-ctl --no-ovs-vswitchd stop
```

### check interface rx-tx status:
```bash
sar -n DEV 1
```

### check pmd status
```bash
ovs-appctl dpif-netdev/pmd-stats-show
ovs-appctl dpif/show
```

# 3. T-Rex Traffic Generator
### 3.1. Config t-rex (packet src,destination IP/MAC)
```bash
- nano /etc/trex_cfg.yaml
```
```bash
  port_info:
      - dest_mac: 40:a6:b7:19:f2:39 # MAC OF port 1 t-rex 
        src_mac:  40:a6:b7:19:f2:38
      - dest_mac: 40:a6:b7:19:f2:38 # MAC OF port 0 t-rex
        src_mac:  40:a6:b7:19:f2:39
```
### 3.2. Start T-gen (1 tab console)
```bash
cd v2.92
./t-rex-64 -i -c 10
```

### 3.3. Generate traffic (1 tab console)
```bash
cd v2.92
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

### Clean up, revert environment After Finishing Benchmark
Master
```
kubectl delete -f ovs-pod.yaml
kubectl delete -f net_attach_definition.yaml
```

Worker
```
ovs-vsctl del-br ovs-br0
/usr/local/share/openvswitch/scripts/ovs-ctl --no-ovsdb-server --db-sock="/usr/local/var/run/openvswitch/db.sock" stop
/usr/local/share/openvswitch/scripts/ovs-ctl --no-ovs-vswitchd stop

root@worker:~/dpdk-stable-22.11.1/usertools# ./dpdk-devbind.py -b i40e 0000:af:00.0 0000:af:00.1
```
