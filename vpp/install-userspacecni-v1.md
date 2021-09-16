## [h2] How to enable IOMMU in CentOS 7 KVM

`http://kkchi5407.myds.me/wordpress/2018/11/15/how-to-enable-iommu-in-centos-7-kvm/`

lscpu |grep virtualization

Refer:
`https://github.com/intel/userspace-cni-network-plugin`

## [h2] Install meson ninja

`https://mesonbuild.com/SimpleStart.html`
sudo dnf install meson ninja-build

yum install epel-release
yum install gcc gcc-c++
yum install meson

## [h2] Install go

1.
`wget https://golang.org/dl/go1.15.2.linux-amd64.tar.gz`
`tar -C /usr/local -xzf go1.15.2.linux-amd64.tar.gz`
2.
`vi ~/.bash_profile`

```
export GOPATH=/usr/local/go/bin
export CNI_PATH=$GOPATH/src/github.com/containernetworking/plugins/bin

PATH=$PATH:$GOPATH
export PATH
```

`source ~/.bash_profile`
3.
`go version`
4.
go version go1.15.2 linux/amd64

## [h2] Install github

1.
`yum install -y git`

## [h2] To get and build the Userspace CNI plugin:

1.

   ```cd $GOPATH/src/
   go get github.com/intel/userspace-cni-network-plugin
   cd github.com/intel/userspace-cni-network-plugin
   make```
2.
`cp userspace/userspace /opt/cni/bin`

[h2] Install DPDK
`https://github.com/openvswitch/ovs/blob/master/Documentation/intro/install/dpdk.rst`
DPDK: 20.11

[h2] Install vpp
Refer: https://github.com/intel/userspace-cni-network-plugin/blob/master/README.md#vpp-cni-library-intro
VPP CNI Library is written in GO and used by UserSpace CNI to interface with the VPP GO-API. When the CNI is invoked, VPP CNI library opens a GO Channel to the local VPP instance and passes gRPC messages between the two.

As mentioned above, to build the Userspace CNI, VPP needs to be installed, or serveral VPP files to compile against. When VPP is installed, it copies it's json API files to /usr/share/vpp/api/. VPP CNI Libary uses these files to compile against and generate the properly versioned messages to the local VPP Instance. So to build the VPP CNI, VPP must be installed (or the proper json files must be in /usr/share/vpp/api/).
   

