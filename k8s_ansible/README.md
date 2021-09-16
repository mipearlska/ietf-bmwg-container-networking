# k8s cluster setup

This project aims to set up and programmatically deploy a Kubernetes cluster on CentOS 7 machines with the help of Kubeadm. It uses ansible and requires very little intervention.

## Getting Started
The following steps aim to describe the minimum required to successfully run this script.


### Prerequisites

Ansible should be installed on the master node. 

Hardware prerequisites as suggested by Kubernetes documentation:

-   Master node's minimal  **required**  memory is 2GB and the worker node needs minimum 1GB.
-   The master node needs at least 1.5 cores and the worker node needs at least 0.7 cores.

Refer: https://phoenixnap.com/kb/how-to-install-kubernetes-on-centos

### Step 1: Configure Kubernetes Repository

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```



### Step 2: Install kubelet, kubeadm, and kubectl, docker

On both master & worker:

```
	sudo yum install -y kubelet kubeadm kubectl
	systemctl enable kubelet
	systemctl start kubelet
	systemctl status kubelet
```

​	

	sudo yum install -y docker
	systemctl enable docker.service
	systemctl start docker
	systemctl status docker

### Step 3: Set Hostname on Nodes
```
sudo hostnamectl set-hostname master-node
sudo hostnamectl set-hostname worker-node1
sudo reboot
```
On both master & worker:	

```
sudo vi /etc/hosts
192.168.26.40 master-node
192.168.26.41 worker-node
```

Step 4: Configure Firewall
On the Master Node enter:
	

```
sudo firewall-cmd --permanent --add-port=6443/tcp
	sudo firewall-cmd --permanent --add-port=2379-2380/tcp
	sudo firewall-cmd --permanent --add-port=10250/tcp
	sudo firewall-cmd --permanent --add-port=10251/tcp
	sudo firewall-cmd --permanent --add-port=10252/tcp
	sudo firewall-cmd --permanent --add-port=10255/tcp
	sudo firewall-cmd --reload
```

​	

	sudo firewall-cmd --list-all

- worker node:
  	

```
sudo firewall-cmd --permanent --add-port=10251/tcp
	sudo firewall-cmd --permanent --add-port=10255/tcp
	sudo firewall-cmd --permanent --add-port=10250/tcp
	firewall-cmd --reload
```

​	

	sudo firewall-cmd --list-all

### Step 5: Update Iptables Settings

- on master node & worker:

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

​	

	sysctl --system

### Step 6: Disable SELinux

- on master node & worker:

```
	sudo setenforce 0
	sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

​	

### (skip) Step 7: Disable SWAP

- On both master and worker:

```
sudo sed -i '/swap/d' /etc/fstab
sudo swapoff -a
```

### ********** Running K8s_ansible **********

Prerequesite:
	

```
sudo yum install sshpass
sudo yum install libselinux-python
#sudo yum install libselinux-python3
pip install selinux
cp -r /usr/lib64/python2.7/site-packages/selinux $VIRTUAL_ENV/lib/python2.7/site-packages
```
### Setup
In order the configure the cluster, we'll need to modify the `hosts` and `env_variable` file. The `hosts` file has the following structure:

```
[master]
master ansible_host=192.168.26.40 ansible_connection=local ansible_user=root ansible_password=123456

[worker]
worker ansible_host=192.168.26.41 ansible_connection=ssh ansible_ssh_user=root ansible_ssh_pass=123456 ansible_ssh_common_args='-o StrictHostKeyChecking=no'

```
In this configuration file, connection details should be filled in. In case more nodes within the cluster are needed, add lines as necessary to the worker group within the `hosts` file.

In the `env_variables` file the following parameters should be entered as per your environment

ad_addr: 192.168.26.40
cidr_v: 10.244.0.0/16

In ad_addr enter your master node advertise ip address and in cidr_v your cidr range for the pods.

### Usage
In order to use the script, download or clone the repository to the root of what will be the master node.

Navigate to its contents and execute the following command as regular user (this will prevent errors throughout configuration and deployment) on whichever machine you wish to use as the master node (this host will be the one running kubectl):

```
ansible-playbook settingup_k8s_cluster.yml
```
You can verify the installation by running:
```
kubectl get nodes
```
And verifying the readiness of the nodes. More information may be obtained with `kubectl describe nodes` if needed.

To clear the cluster, execute the following command
ansible-playbook clear_k8s_cluster.yml

### Debugging
In case a step goes wrong within the installation, ansible should display a message, however, there's also files to debug if the installation had something to do within k8s. In the case of the master node, we should be able to find a `join_token.txt` with necessary logs. On worker nodes, the relevant file is `node_joined.txt`.
​	

	ssh-key-gen
	ssh-copy-id worker@192.168.26.41
	
	cd /root/k8s_ansible
	ansible-playbook installation_k8s.yml
	ansible-playbook settingup_k8s_cluster.yml



**Source code from ViNePerf project: https://wiki.anuket.io/display/HOME/ViNePERF**
