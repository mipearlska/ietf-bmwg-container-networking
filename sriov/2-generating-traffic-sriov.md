# L2fwd
`docker pull huyng14/dpdkapp-v19.08`

```
[root@sriov-pod /]#
 0000:af:02.0 -w 0000:af:0a.0 -- -p 0x3 -T 120 --no-mac-updatinget-mem=0,1024 -w
ENTER dpdk-app:
 argc=18
 l2fwd -n 4 -l 38-39 --master-lcore 38 --socket-mem=0,1024 -w 0000:af:02.0 -w 0000:af:0a.0 -- -p 0x3 -T 120 --no-mac-updating
EAL: Detected 80 lcore(s)
EAL: Detected 2 NUMA nodes
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: Probing VFIO support...
EAL: VFIO support initialized
EAL: PCI device 0000:af:02.0 on NUMA socket 1
EAL:   probe driver: 8086:154c net_i40e_vf
EAL:   using IOMMU type 1 (Type 1)
EAL: PCI device 0000:af:0a.0 on NUMA socket 1
EAL:   probe driver: 8086:154c net_i40e_vf
MAC updating disabled
Lcore 38: RX port 0
Lcore 39: RX port 1
Initializing port 0... done:
Port 0, MAC address: 22:CD:67:1E:C2:5F

Initializing port 1... done:
Port 1, MAC address: A2:E4:F8:FE:9B:1E


Checking link statusdone
Port0 Link Up. Speed 40000 Mbps - full-duplex
Port1 Link Up. Speed 40000 Mbps - full-duplex
L2FWD: entering main loop on lcore 38
L2FWD:  -- lcoreid=38 portid=0

Port statistics ====================================
Statistics for port 0 ------------------------------
Packets sent:                        0
Packets received:                    0
Packets dropped:                     0
Statistics for port 1 ------------------------------
Packets sent:                        0
Packets received:                    0
Packets dropped:                     0
Aggregate statistics ===============================
Total packets sent:                  0
Total packets received:              0
Total packets dropped:               0
====================================================
```

## Note:

- ### core and socket-mem on same NUMA

# /etc/trex_cfg.yaml

```
[root@tgen ~]# cat /etc/trex_cfg.yaml
- version         : 2
  interfaces      : ["86:00.0","86:00.1"]
  port_limit      : 2
  enable_zmq_pub  : true  # enable publisher for stats data
  c               : 4
# for system of 1Gb/sec NIC or VM enable this
  port_bandwidth_gb : 40  # port bandwidth 10Gb/sec , for VM put here 1 for XL710 put 40
  platform :
        master_thread_id  : 0
        latency_thread_id : 5
        dual_if   :
             - socket   : 0
               threads  : [1,2,3,4,6,7,8,9,10,11]
  port_info       :  # set eh mac addr
          - dest_mac        :  22:CD:67:1E:C2:5F        # port 0, in case SR-IOV, destmac is VNF's mac
            src_mac         :  A2:E4:F8:FE:9B:1E
          - dest_mac        :  A2:E4:F8:FE:9B:1E        # port 1
            src_mac         :  22:CD:67:1E:C2:5F

#  port_info
#         - dest_mac        : 40:a6:b7:19:f2:39			# 40:a6:b7:19:f2:39 MAC's port 1 on traffic gen
#           src_mac         : 40:a6:b7:19:f2:38         # 40:a6:b7:19:f2:38 MAC's port 0 on traffic gen
#         - dest_mac        : 40:a6:b7:19:f2:38
#           src_mac         : 40:a6:b7:19:f2:39
```

# C1.Running l2fwd

```
l2fwd -n 4 -l 38-39 --master-lcore 38 --socket-mem=0,1024 -w 0000:af:02.0 -w 0000:af:0a.0 -- -p 0x3 -T 120 --no-mac-updating
```

- checking VF running for the POD by

```
[root@worker usertools]# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether a4:bf:01:3e:22:e3 brd ff:ff:ff:ff:ff:ff
3: eno2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether a4:bf:01:3e:22:e4 brd ff:ff:ff:ff:ff:ff
4: enp175s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 3c:fd:fe:ec:4c:88 brd ff:ff:ff:ff:ff:ff
    vf 0 MAC 22:cd:67:1e:c2:5f, spoof checking off, link-state auto, trust on
    vf 1 MAC aa:65:11:0d:ef:b6, spoof checking on, link-state auto, trust off
5: enp175s0f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 3c:fd:fe:ec:4c:89 brd ff:ff:ff:ff:ff:ff
    vf 0 MAC a2:e4:f8:fe:9b:1e, spoof checking off, link-state auto, trust on
    vf 1 MAC aa:9e:af:44:85:29, spoof checking on, link-state auto, trust off
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:b7:ab:4c:17 brd ff:ff:ff:ff:ff:ff
7: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default
    link/ether 7a:af:8f:de:e3:bb brd ff:ff:ff:ff:ff:ff
8: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 46:d1:63:3e:98:1c brd ff:ff:ff:ff:ff:ff
18: veth40a7b157@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP mode DEFAULT group default
    link/ether 56:fe:9f:f2:8a:87 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

```
> vf 0 MAC 22:cd:67:1e:c2:5f, spoof checking off, link-state auto, trust on
> vf 0 MAC a2:e4:f8:fe:9b:1e, spoof checking off, link-state auto, trust on
> are running as interfaces on POD
```


# C2.Generating traffic

- Configure DPDK running vfio-pci driver
``` bash
sudo yum install -y kernel-devel-`uname -r`
sudo yum install -y kernel
sudo yum install -y kernel-devel 
sudo yum group install -y "Development tools"
sudo reboot

tar -zxvf i40e-2.14.13.tar.gz
cd i40e-2.14.13.tar.gz/src
make
make install
modprobe i40e
lsmod | grep i40e
modinfo i40e | grep ver
modprobe vfio-pci

./dpdk-20.11/usertools/dpdk-devbind.py -s
./dpdk-20.11/usertools/dpdk-devbind.py -b vfio-pci 0000:86:00.0 0000:86:00.1
```

> ```
> Output example
> [root@tgen ~]# ./dpdk-20.11/usertools/dpdk-devbind.py -s
> 
> Network devices using DPDK-compatible driver
> ============================================
> 
> 0000:86:00.0 'Ethernet Controller XL710 for 40GbE QSFP+ 1583' drv=vfio-pci unused=i40e
> 0000:86:00.1 'Ethernet Controller XL710 for 40GbE QSFP+ 1583' drv=vfio-pci unused=i40e
> 
> Network devices using kernel driver
> ===================================
> 
> 0000:3d:00.0 'Ethernet Connection X722 for 10GBASE-T 37d2' if=eno1 drv=i40e unused=vfio-pci *Active*
> 0000:3d:00.1 'Ethernet Connection X722 for 10GBASE-T 37d2' if=eno2 drv=i40e unused=vfio-pci
> ```

- Configure hugepages

  ```bash
  cat /etc/default/grub
  GRUB_TIMEOUT=5
  GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
  GRUB_DEFAULT=saved
  GRUB_DISABLE_SUBMENU=true
  GRUB_TERMINAL_OUTPUT="console"
  GRUB_CMDLINE_LINUX="crashkernel=auto spectre_v2=retpoline rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet intel_iommu=on default_hugepagesz=1G hugepagesz=1G hugepages=8 hugepagesz=2M hugepages=4096"
  GRUB_DISABLE_RECOVERY="true"
  ```

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
reboot
```

- Configure TREX:

```
wget https://trex-tgn.cisco.com/trex/release/v2.37.tar.gz
vi /etc/trex_cfg.yaml
```


```bash
./t-rex-64 -i -c 4

*** In another shell:
./opt/trex/v2.82/trex-console
tui
start -f stl/bench.py -t size=1518 -p 0 -m 100% --force -d 20
start -f stl/bench.py -t size=64 -p 0 -m 100% --force -d 20

Cause corruption testpmd && trex	
	start -f stl/imix.py -p 0 -m 100% --force
```

Refer: 
	https://trex-tgn.cisco.com/trex/doc/trex_stateless_bench.html



