# Hardware Specification

![topology](images/topology.png)

|     Node name     | Specification                                                | Description                                                  |
| :---------------: | :----------------------------------------------------------- | :----------------------------------------------------------- |
|    Master node    | - CPU : E5-2620 v3 @2.40GHz; 12 cores <br/>- MEM : 32GB <br/>- DISK : 1TB <br/>- NIC : control plane 1G <br/>- OS : CentOS Linux 7.9 | \- Container Deployment and Network Allocation<br/>\- Kubernetes Master |
|    Worker node    | - CPU : Gold 6148 CPU @2.40GHz; 80 cores <br/>- MEM : 256GB<br/>- DISK : 2TB<br/>- NIC : data plane XL710 for 40GbE <br/>- OS : CentOS Linux 7.9 | \- Container Service<br/>\- Kubernets Worker<br/>- System Under Test (SUT) |
| Traffic Generator | - CPU : Gold 6148 CPU @2.40GHz; 80 cores<br/>- MEM : 256GB<br/>- DISK : 2TB<br/>- NIC : data plane XL710 for 40GbE<br/>- OS : CentOS Linux 7.9 | \- Traffic Generator<br/>\- TREX v2.82                       |

## Software Prerequisites for Master Nodes, and Worker Nodes

```
Step 1: Installing env packages for CentOS
	sudo yum install epel-release
	sudo yum install python-pip
	pip --version

Step 2: Installing Ansible by "virtual environments"
	pip install --upgrade pip
	pip install virtualenv
	
	pip uninstall ansible
	python -m virtualenv ansible
	source ansible/bin/activate 
	python -m pip install ansible==2.9.14
```



```
sudo yum install sshpass
sudo yum install libselinux-python
#sudo yum install libselinux-python3
pip install selinux
cp -r /usr/lib64/python2.7/site-packages/selinux $VIRTUAL_ENV/lib/python2.7/site-packages

ssh-key-gen
ssh-copy-id worker@192.168.26.41

cd /root/k8s_ansible
ansible-playbook installation_k8s.yml
ansible-playbook settingup_k8s_cluster.yml
```


