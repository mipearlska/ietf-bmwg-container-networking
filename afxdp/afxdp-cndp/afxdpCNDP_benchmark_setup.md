# Prerequisites
### OS: Ubuntu 22.04 (OS kernel 5.15)
### Install docker, kubernetes v1.23.5, Multus CNI, AFXDP-k8S Plugin (daemonset below - apply will install plugin and CNI bin to all nodes)
### Setup hugepages 2Mi at worker node
### Build CNDP image (disable MAC_SWAP function in cndp/examples/cndpfwd/main.h before build, replace the default cndp/examples/cndpfwd/fwd.jsonc (below), then copy the Dockerfile to /cndp folder)
```
https://github.com/CloudNativeDataPlane/cndp/tree/main/containerization/docker
```

```bash
apt install -y iproute2
apt install -y net-tools
```
# Benchmarking Flow
0. (t-rex) Config t-rex traffic profile (/etc/trex_cfg.yaml) as below then start t-rex. Not send any traffic yet
1. (master) Apply afxdp plugin daemonsets from master (check kube-afxdp-device-plugin Running at both nodes and afxdp cni bin inside /opt/cni/bin at both nodes)
2. (master) Apply network attachment definition (check kubectl get net-attach-def)
3. (worker) Config NIC interface at DUT worker node (promisc, combined 1)
4. (master) Deploy pod (build cndp pod image with proper fwd.jsonc configuration first if not done yet)
5. (master) Kubectl exec into pod then run cndpfwd app
6. (t-rex) Send traffic from t-rex
7. (t-rex) Run benchmarking NDR app from t-rex

# 1. Set up AFXDP-k8S Plugin (master node)
```
https://github.com/intel/afxdp-plugins-for-kubernetes
```

### Daemonset (replace the default deployments/daemonset.yml in afxdp plugin github with worker node interfaces configuration)
From master
```bash
kubectl apply -f daemonset.yml
```
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: afxdp-dp-config
  namespace: kube-system
data:
  config.json: |
    {
       "logLevel":"debug",
       "logFile":"afxdp-dp.log",
       "pools":[
          {
             "name":"myPool1",
             "mode":"primary",
             "drivers":[
                {
                   "name":"i40e",
                   "primary":1,
                   "excludeDevices":[
                      {
                         "name":"eno1"
                      },
                      {
                         "name":"eno2"
                      },
                      {
                         "name":"enp175s0f1"
                      }
                   ]
                }
             ]
          },
          {
             "name":"myPool2",
             "mode":"primary",
             "drivers":[
                {
                   "name":"i40e",
                   "primary":1,
                   "excludeDevices":[
                      {
                         "name":"eno1"
                      },
                      {
                         "name":"eno2"
                      },
                      {
                         "name":"enp175s0f0"
                      }
                   ]
                }
             ]
          }
       ]
    }
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: afxdp-device-plugin
  namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-afxdp-device-plugin
  namespace: kube-system
  labels:
    tier: node
    app: afxdp
spec:
  selector:
    matchLabels:
      name: afxdp-device-plugin
  template:
    metadata:
      labels:
        name: afxdp-device-plugin
        tier: node
        app: afxdp
    spec:
      hostNetwork: true
      nodeSelector:
        kubernetes.io/arch: amd64
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      serviceAccountName: afxdp-device-plugin
      containers:
        - name: kube-afxdp
          image: intel/afxdp-plugins-for-kubernetes:latest
          imagePullPolicy: IfNotPresent
          securityContext:
            capabilities:
              drop:
                - all
              add:
                - SYS_ADMIN
                - NET_ADMIN
          resources:
            requests:
              cpu: "250m"
              memory: "40Mi"
            limits:
              cpu: "1"
              memory: "200Mi"
          volumeMounts:
            - name: unixsock
              mountPath: /tmp/afxdp_dp/
            - name: devicesock
              mountPath: /var/lib/kubelet/device-plugins/
            - name: resources
              mountPath: /var/lib/kubelet/pod-resources/
            - name: config-volume
              mountPath: /afxdp/config
            - name: log
              mountPath: /var/log/afxdp-k8s-plugins/
            - name: cnibin
              mountPath: /opt/cni/bin/
      volumes:
        - name: unixsock
          hostPath:
            path: /tmp/afxdp_dp/
        - name: devicesock
          hostPath:
            path: /var/lib/kubelet/device-plugins/
        - name: resources
          hostPath:
            path: /var/lib/kubelet/pod-resources/
        - name: config-volume
          configMap:
            name: afxdp-dp-config
            items:
              - key: config.json
                path: config.json
        - name: log
          hostPath:
            path: /var/log/afxdp-k8s-plugins/
        - name: cnibin
          hostPath:
            path: /opt/cni/bin/
```

### Network Attachment Definition
From master
```bash
kubectl apply -f network-attachment-definition1.yaml
kubectl apply -f network-attachment-definition2.yaml
```
```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: afxdp-network1
  annotations:
    k8s.v1.cni.cncf.io/resourceName: afxdp/myPool1
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "afxdp",
      "mode": "primary",
      "logFile": "afxdp-cni.log",
      "logLevel": "debug",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.1.0/24",
        "rangeStart": "192.168.1.200",
        "rangeEnd": "192.168.1.220",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.1.1"
      }
    }'
```

```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: afxdp-network2
  annotations:
    k8s.v1.cni.cncf.io/resourceName: afxdp/myPool2
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "afxdp",
      "mode": "primary",
      "logFile": "afxdp-cni.log",
      "logLevel": "debug",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.1.0/24",
        "rangeStart": "192.168.1.200",
        "rangeEnd": "192.168.1.220",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.1.1"
      }
    }'

```

```bash
kubectl get node worker41 -o json | jq '.status.allocatable'
```
```
{
  "afxdp/myPool": "0",
  "afxdp/myPool1": "1",
  "afxdp/myPool2": "1",
  "cpu": "80",
  "ephemeral-storage": "94580335255",
  "hugepages-1Gi": "16Gi",
  "hugepages-2Mi": "4Gi",
  "memory": "242740388Ki",
  "pods": "110"
}
```

### Configure NIC interface
From worker
```bash
ifconfig enp175s0f0 promisc
ifconfig enp175s0f1 promisc
ethtool -L enp175s0f0 combined 1
ethtool -L enp175s0f1 combined 1
ethtool -l enp175s0f0 #check
ethtool -g enp175s0f0 #check
```
Turn off Promisc mode after test:
```bash
ifconfig enp175s0f0 -promisc
```
# 3. CNDPFWD application (master node)

### POD yaml
From master
Image: 
```
docker pull mipearlska/cndp
```
This image has been build with our system fwd.jsonc configuration (please change configuration inside fwd.jsonc suitable to yours)

```
apiVersion: v1
kind: Pod
metadata:
  name: cndp-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: afxdp-network1, afxdp-network2
  labels:
    app: cndp
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
    - name: cndp-0
      command: ["sleep", "infinity"]
#      command: ["/bin/bash"]
#      args: ["while true; do sleep 30; done;" ]
      image: mipearlska/cndp:latest
      imagePullPolicy: Never
      securityContext:
        privileged: true
      resources:
        requests:
          afxdp/myPool1: '1'
          afxdp/myPool2: '1'
          hugepages-2Mi: 512Mi
          memory: 2Gi
        limits:
          afxdp/myPool1: '1'
          afxdp/myPool2: '1'
          hugepages-2Mi: 512Mi
          memory: 2Gi
      volumeMounts:
        - name: shared-data
          mountPath: /var/run/cndp/

```

### CNDP fwd.jsonc configuration file (config suitable lcore-group, lports, thread)
Replace the default fwd.jsonc in /cndp/examples/cndpfwd/fwd.jsonc
Must be configured before building the cndp image
```
{
    // (R) - Required entry
    // (O) - Optional entry
    // All descriptions are optional and short form is 'desc'
    // The order of the entries in this file are handled when it is parsed and the
    // entries can be in any order.

    // (R) Application information
    //    name        - (O) the name of the application
    //    description - (O) the description of the application
    "application": {
        "name": "cndpfwd",
        "description": "A simple packet forwarder for pktdev and xskdev"
    },

    // (O) Default values
    //    bufcnt - (O) UMEM default buffer count in 1K increments
    //    bufsz  - (O) UMEM buffer size in 1K increments
    //    rxdesc - (O) Number of RX ring descriptors in 1K increments
    //    txdesc - (O) Number of TX ring descriptors in 1K increments
    //    cache  - (O) MBUF Pool cache size in number of entries
    //    mtype  - (O) Memory type for mmap allocations
    "defaults": {
        "bufcnt": 16,
        "bufsz": 2,
        "rxdesc": 2,
        "txdesc": 2,
        "cache": 256,
        "mtype": "2MB"
    },

    // List of all UMEM's to be created
    // key/val - (R) The 'key' is the name of the umem for later reference.
    //               The 'val' is the object describing the UMEM buffer.
    //               Multiple umem regions can be defined.
    // A UMEM can support multiple lports using the regions array. Each lports can use
    // one of the regions.
    //    bufcnt  - (R) The number of buffers in 1K increments in the UMEM space.
    //    bufsz   - (R) The size in 1K increments of each buffer in the UMEM space.
    //    mtype   - (O) If missing or empty string or missing means use 4KB or default system pages.
    //    regions - (O) Array of sizes one per region in 1K increments, total must be <= bufcnt
    //    rxdesc  - (O) Number of RX descriptors to be allocated in 1K increments,
    //                  if not present or zero use defaults.rxdesc, normally zero.
    //    txdesc  - (O) Number of TX descriptors to be allocated in 1K increments,
    //                  if not present or zero use defaults.txdesc, normally zero.
    //    shared_umem - (O) Set to true to use xsk_socket__create_shared() API, default false
    //    description | desc - (O) Description of the umem space.
    "umems": {
        "umem0": {
            "bufcnt": 32,
            "bufsz": 2,
            "mtype": "2MB",
            "regions": [
                16,
                16
            ],
            "rxdesc": 0,
            "txdesc": 0,
            "description": "UMEM Description 0"
        }
    },

    // List of all lports to be used in the application
    // An lport is defined by a netdev/queue ID pair, which is a socket containing a Rx/Tx ring pair.
    // Each queue ID is assigned to a single socket or a socket is the lport defined by netdev/qid.
    // Note: A netdev can be shared between lports as the qid is unique per lport
    //       If netdev is not defined or empty then it must be a virtual interface and not
    //       associated with a netdev/queue ID.
    // key/val - (R) The 'key' is the logical name e.g. 'eth0:0', 'eth1:0', ... to be used by the
    //               application to reference an lport. The 'val' object contains information about
    //               each lport.
    //    netdev        - (R) The netdev device to be used, the part before the colon
    //                     must reflect the netdev name
    //    pmd           - (R) All PMDs have a name i.e. 'net_af_xdp', 'ring', ...
    //    qid           - (R) Is the queue id to use for this lport, defined by ethtool command line
    //    umem          - (R) The UMEM assigned to this lport
    //    region        - (O) UMEM region index value, default region 0
    //   enp175s0f0 busy_poll     - (O) Enable busy polling support, true or false, default false
    //    busy_polling  -     Same as above
    //    busy_timeout  - (O) 1-65535 or 0 - use default value, values in milliseconds
    //    busy_budget   - (O) 0xFFFF disabled, 0 use default, >0 budget value
    //    unprivileged  - (O) inhibit loading the BPF program if true, default false
    //    force_wakeup  - (O) force TX wakeup calls for CVL NIC, default false
    //    skb_mode      - (O) Enable XDP_FLAGS_SKB_MODE when creating af_xdp socket, forces copy mode, default false
    //    description   - (O) the description, 'desc' can be used as well
    "lports": {
        "enp175s0f0:0": {
            "pmd": "net_af_xdp",
            "qid": 0,
            "umem": "umem0",
            "region": 0,
            "description": "LAN 0 port"
        },
        "enp175s0f1:0": {
            "pmd": "net_af_xdp",
            "qid": 0,
            "umem": "umem0",
            "region": 1,
            "description": "LAN 1 port"
        }
    },

    // (O) Define the lcore groups for each thread to run
    //     Can be integers or a string for a range of lcores
    //     e.g. [10], [10-14,16], [10-12, 14-15, 17-18, 20]
    // Names of a lcore group and its lcores assigned to the group.
    // The initial group is for the main thread of the application.
    // The default group is special and is used if a thread if not assigned to a group.
    "lcore-groups": {
        "initial": [10],
        "group0": [13],
        "group1": [14],
        "default": ["15-16"]
    },

    // (O) Set of common options application defined.
    //     The Key can be any string and value can be boolean, string, array or integer
    //     An array must contain only a single value type, boolean, integer, string and
    //     can't be a nested array.
    //   pkt_api    - (O) Set the type of packet API xskdev or pktdev
    //   no-metrics - (O) Disable metrics gathering and thread
    //   no-restapi - (O) Disable RestAPI support
    //   cli        - (O) Enable/Disable CLI supported
    //   mode       - (O) Mode type [drop | rx-only], tx-only, [lb | loopback], fwd, acl-strict, acl-permissive
    //   uds_path   - (O) Path to unix domain socket to get xsk map fd
    "options": {
        "pkt_api": "xskdev",
        "no-metrics": false,
        "no-restapi": false,
        "cli": true,
        "mode": "drop"
    },

    // List of threads to start and information for that thread. Application can start
    // it's own threads for any reason and are not required to be configured by this file.
    //
    //   Key/Val   - (R) A unique thread name.
    //                   The format is <type-string>[:<identifier>] the ':' and identifier
    //                   are optional if all thread names are unique
    //      group  - (O) The lcore-group this thread belongs to. The
    //      lports - (O) The list of lports assigned to this thread and can not shared lports.
    //      description | desc - (O) The description
    "threads": {
        "main": {
            "group": "initial",
            "description": "CLI Thread"
        },
        "fwd:0": {
            "group": "group0",
            "lports": ["enp175s0f0:0"],
            "description": "Thread 0"
        },
        "fwd:1": {
            "group": "group1",
            "lports": ["enp175s0f1:0"],
            "description": "Thread 1"
        }
    }
}

```

### Running l2fwd application
```bash
kubectl exec -it cndp-pod /bin/bash
cndpfwd -c fwd.jsonc fwd
```

# 2. T-Rex Traffic Generator
### 2.1. Config t-rex (packet src,destination IP/MAC)
```bash
- nano /etc/trex_cfg.yaml
```
```bash
  port_info:
      - dest_mac: 40:a6:b7:19:f2:01 # MAC OF port 1 t-rex 
        src_mac:  40:a6:b7:19:f2:00
      - dest_mac: 40:a6:b7:19:f2:00 # MAC OF port 0 t-rex
        src_mac:  40:a6:b7:19:f2:01
```
### 2.2. Start T-gen (1 tab console)
```bash
cd v2.92
./t-rex-64 -i -c 10
```

### 2.3. Generate traffic (1 tab console)
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

### 2.4. Benchmark application run (No Drop Rate Benchmarking) - (Manual: https://trex-tgn.cisco.com/trex/doc/trex_ndr_bench_doc.html)
```bash
./ndr --stl --port 0 1 -v --profile stl/bench.py --prof-tun size=1518 --opt-bin-search
```


###Others
```
docker save -o cndp.tar cndp:latest
scp -r cndp.tar worker2@192.168.26.42:/home/worker2
docker load -i cndp.tar
```
