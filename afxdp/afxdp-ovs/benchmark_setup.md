# Prerequisites
### OS: Ubuntu 22.04 (OS kernel 5.15)
### Install docker, kubernetes v1.23.5, Multus CNI, Userspace CNI
### Install DPDK 21.11.1 (use meson ninja)
### Install OVS-DPDK 2.17.2 (current latest) (enable afxdp support flag when run ..configure)

```bash
sudo apt-get install -qy kubelet=1.23.5-00 kubectl=1.23.5-00 kubeadm=1.23.5-00
```

```bash
install libbpf (apt install libbpf-dev)
install libtool (apt-get install libtool)
install autoconf (apt-get install autoconf)
git clone https://github.com/openvswitch/ovs.git
cd ovs
./boot.sh
./configure --with-dpdk=static --enable-afxdp
make -j $(nproc)
sudo make install
```

# 1. T-Rex Traffic Generator
### 1.1. Config t-rex (packet src,destination IP/MAC)
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
### 1.2. Start T-gen (1 tab console)
```bash
cd v2.92
./t-rex-64 -i -c 10
```

### 1.3. Generate traffic (1 tab console)
```bash
cd v2.92
./trex-console
tui
```

### Generate test traffic
```bash
start -f stl/bench.py -p 0 -t size=1518
start -f stl/bench.py -p 0 -m 100% --force -t size=1518
start -f stl/bench.py -p 1 -m 100% --force -t size=1518
start -f stl/bench.py -m 100% --force -t size=1518
```
(p = from port 0 or 1 of tgen, no -p flag = generate from both ports, -size = packet size, -m = %max line rate of NIC card)

### 1.4. Benchmark application run (No Drop Rate Benchmarking) - (Manual: https://trex-tgn.cisco.com/trex/doc/trex_ndr_bench_doc.html)
```bash
./ndr --stl --port 0 1 -v --profile stl/bench.py --prof-tun size=1518 --opt-bin-search
```

# 2. Set up packet flow through OVS-DPDK
```bash
ethtool -L enp175s0f0 combined 1
ethtool -L enp175s0f1 combined 1
ethtool -G enp175s0f0 rx 4096 tx 4096
ethtool -G enp175s0f1 rx 4096 tx 4096
ethtool -l enp175s0f0 #check
ethtool -g enp175s0f0 #check
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

add OVS port (2 afxdp ports, 2 vhostuser ports)
```bash
ovs-vsctl add-port ovs-br0 enp175s0f0 -- set interface enp175s0f0 type="afxdp" options:n_rxq=1 other_config:pmd-rxq-affinity="0:4"
ovs-vsctl add-port ovs-br0 enp175s0f1 -- set interface enp175s0f1 type="afxdp" options:n_rxq=1 other_config:pmd-rxq-affinity="0:6"
ovs-vsctl add-port ovs-br0 dpdkvhostuser0 -- set Interface dpdkvhostuser0 type=dpdkvhostuser
ovs-vsctl add-port ovs-br0 dpdkvhostuser1 -- set Interface dpdkvhostuser1 type=dpdkvhostuser
```

check ovs-bro bridge status
```bash
ovs-vsctl show
ovs-ofctl show ovs-br0
```

add traffic routing flows between afxdp and vhostuser ports
```bash
ovs-ofctl --timeout 10 -O Openflow13 add-flow ovs-br0 in_port=1,idle_timeout=0,action=output:3
ovs-ofctl --timeout 10 -O Openflow13 add-flow ovs-br0 in_port=3,idle_timeout=0,action=output:1
ovs-ofctl --timeout 10 -O Openflow13 add-flow ovs-br0 in_port=2,idle_timeout=0,action=output:4
ovs-ofctl --timeout 10 -O Openflow13 add-flow ovs-br0 in_port=4,idle_timeout=0,action=output:2
ovs-ofctl dump-flows ovs-br0
```

# 3. L2FWD application

### Userspace CNI Network Attachment Definition

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
    image: dpdk-app-centos:latest
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
        memory: 2Mi
        hugepages-2Mi: 512Mi
      limits:
        hugepages-2Mi: 512Mi
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
```bash
cd $DPDK_DIRECTORY/build/examples

./dpdk-l2fwd -n 4 -l 15-18 --single-file-segments --vdev=virtio_user0,path=/usr/local/var/run/openvswitch/dpdkvhostuser0 --vdev=virtio_user1,path=/usr/local/var/run/openvswitch/dpdkvhostuser1 --no-pci -- -p 0x3 -T 10 --no-mac-updating
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