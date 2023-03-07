## Testbed Topology

![topology](00_images/topo.png)

## Benchmarking Configuration

|     Node name     | Specification                                                | Description                                                  |
| :---------------: | :----------------------------------------------------------- | :----------------------------------------------------------- |
|    Worker node hardware   | - CPU : Intel(R) Xeon(R) Gold 5220R CPU @ 2.20GHz; 80 cores <br/>- MEM : 256GB<br/>- DISK : 2TB<br/>- NIC : XL710 for 40GbE | 
|    Worker node software  | - OS : Ubuntu 22.04 (18.04 for VPP test) <br/>- DPDK : v22.11.1 <br/>- OVS: v3.1.90 (DPDK, AFXDP enabled)<br/>- VPP : v19.04 <br/>- Docker : v20.10.22 <br/>- Kubernetes : v1.23.5 <br/>- K8s CNI : Flannel, Multus, Userspace, SRIOV, AFXDP | 
| Traffic Generator | - Software : T-Rex v2.92, v2.73 <br/>- Benchmarking method : T-Rex Non Drop Rate (ndr) | 





