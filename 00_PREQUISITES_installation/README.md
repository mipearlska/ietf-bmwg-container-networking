# Prequisites Installation at DUT
Our testbed DUT currently uses Ubuntu 22.04
### Libraries, go, hugepages, iommu, docker, kubernetes
### dpdk, VPP, OVS-DPDK, OVS-DPDK-afxdp, SRIOV CNI, userspace CNI, multus CNI
### Built CNI bins (userspace, sriov, afxdp) in 1_cnibin folder
### For AFXDP benchmarking, libbpf is required (need Ubuntu version from 20.04)

# Upgrade kernel to 5.4 (if using Ubuntu 18.04 LTS)
```
sudo apt-get install --install-recommends linux-generic-hwe-18.04
reboot
```

# Libraries (libbpf require ubuntu 20.04 or higher)
```
apt install build-essential
apt install python3-pyelftools
apt install libtool
apt install libbpf
apt install pkg-config
apt-get install -y libnuma-dev
apt install meson
```
For Ubuntu 18.04 or lower, meson version is 0.45. Need to uninstall current one and install higher version
```
sudo apt-get remove meson
apt install python3-pip
pip3 install meson

Add to the .profile file:
export PATH=$PATH:/usr/local/bin/meson

source .profile
```

# Go 1.19.6
https://www.digitalocean.com/community/tutorials/how-to-install-go-on-ubuntu-20-04

# Setup hugepages-iommu for dpdk
```
nano /etc/default/grub
```
Replace inside /etc/default/grub
```
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt default_hugepagesz=1G hugepagesz=1G hugepages=16"
GRUB_DISABLE_RECOVERY="true"
```
```
grub-mkconfig -o /boot/grub/grub.cfg
reboot
```
Verify
```
cat /proc/meminfo
dmesg |grep -E "DMAR|IOMMU"
```

# Install VPP 19.04 for building userspaceCNI
### (Only Ubuntu 18.04 LTS)  (higher Ubuntu version (20.04, 22.04) cannot build)
``` 
curl -s https://packagecloud.io/install/repositories/fdio/1904/script.deb.sh | sudo bash
sudo apt-get update
sudo apt-get install vpp vpp-plugin-core vpp-plugin-dpdk vpp-dbg vpp-dev vpp-ext-deps vpp-api-python vpp-api-java
```

### Verify
```
vppctl
```
### uninstall
```
sudo apt-get remove --purge vpp*
sudo apt autoremove
```

# Build UserspaceCNI (Requirement: VPP 19.04)
### (Only Ubuntu 18.04 LTS)  (higher Ubuntu version (20.04, 22.04) cannot build)
### In case of using Ubuntu 20.04, 22.04 for benchmarking OVS-afxdp (libbpf only support from Ubuntu 20.04), build the CNI bin first then copy to the higher Ubuntu DUT
### userspace CNI bin must be copied to all nodes (/opt/cni/bin)
```
go get github.com/intel/userspace-cni-network-plugin
```
From $GOPATH
```
cd pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d
go get git.fd.io/govpp.git/cmd/binapi-generator@v0.3.5
make
cp userspace/userspace /opt/cni/bin
```
Sucessful make logs
```
for package in cnivpp/api/* ; do cd $package ; pwd ; go generate ; cd - ; done
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d/cnivpp/api/bridge
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d/cnivpp/api/infra
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d/cnivpp/api/interface
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d/cnivpp/api/memif
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d/cnivpp/api/vhostuser
/root/go/pkg/mod/github.com/intel/userspace-cni-network-plugin@v0.0.0-20220920105437-72003eef8a7d
github.com/intel/userspace-cni-network-plugin/cnivpp/bin_api/memif
github.com/intel/userspace-cni-network-plugin/cnivpp/api/memif
github.com/intel/userspace-cni-network-plugin/cnivpp
github.com/intel/userspace-cni-network-plugin/userspace
```

# Build SRIOV CNI
### SRIOV CNI bin must be copied to all nodes (/opt/cni/bin)
```
git clone https://github.com/k8snetworkplumbingwg/sriov-cni.git
cd sriov-cni
make
cp build/sriov /opt/cni/bin
```
# Install dpdk
```
wget https://fast.dpdk.org/rel/dpdk-22.11.1.tar.xz
tar xf dpdk-22.11.1.tar.xz
cd dpdk-stable-22.11.1
meson build
ninja -C build
sudo ninja -C build install
sudo ldconfig
pkg-config --modversion libdpdk
```

# Install OVS
```
apt install libtool
sudo apt-get install autoconf
apt install libbpf-dev
git clone https://github.com/openvswitch/ovs.git
cd ovs
./boot.sh
./configure --with-dpdk=static --enable-afxdp
make -j $(nproc)
sudo make install
```

ovs-vswitchd --version should return
```
ovs-vswitchd --version

ovs-vswitchd (Open vSwitch) 3.1.90
DPDK 22.11.1
```

# Docker 20.10.22 
Previous steps until "Installing Docker engine" same as official guide
```
sudo apt-get install docker-ce=5:20.10.22~3-0~ubuntu-jammy docker-ce-cli=5:20.10.22~3-0~ubuntu-jammy containerd.io docker-buildx-plugin docker-compose-plugin
```

# Kubernetes 1.23.5
```
sudo apt-get install -qy kubelet=1.23.5-00 kubectl=1.23.5-00 kubeadm=1.23.5-00
```

# Create K8s cluster - flannel CNI

# Install MultusCNI from master
```
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git && cd multus-cni
cat ./deployments/multus-daemonset.yml | kubectl apply -f -

###Veryfying
kubectl get pods --all-namespaces | grep -i multus
```
