# pi_k8s_setup

Setup a Kubernetes cluster on raspberry pi4 with Ansible playbooks.

## Table of contents

* [Prerequisites](#prerequisites)
* [Setup pis](#technologies)
* [k8s_cluster_setup](#k8s_cluster_setup)

## Prerequisites

* Minimum requirements:
* 2CPU cores 2G memory.
* OS Ubuntu 20.04 Focal and Update your release to Jammy.
* Disk space 20GB.

## Setup pis, Users and Settings

### Prepare the users

* Find your pis on the local network with ```nmap -sn -v <192.168.0.0/24>``` in the respective network.
* You may choose to use the ubuntu user or create another user to be utilised as Ansible-user.
[More](https://www.alibabacloud.com/blog/managing-system-users-using-ansible_593861)
* In general you need a user being able to ssh to pis and a file in ```/etc/sudoers.d/file```, with content ```your-user ALL=(ALL) NOPASSWD: ALL``` for Ansible to be able to tinker inside the servers.
* Adjust the the inventory git-prep as needed, by using the findings from `nmap`.
* Adjust the file ansible_cfg and rename to ansible.cfg.
* Test your connection to pis ssh.

```sh
ansible all -m ping -i git-prep
```

* You need to get a pong and success response. If you did not get a pong and success read this instruction again or check the ansible.cfg for details.

### Env

* Create a directory for your playbooks.

```sh
mkdir ~/plays
```

* Download the files in a directory called $HOME/plays.
* You should have a directory structure ($HOME/plays/playbooks) and ($HOME/plays/pi_files/netplan).

### Static addresses

### Before you begin

* ! The sources containing the netplan configurations for the playbooks are in files server(1,2,3), adjust them before running the playbooks.
* Check if the DHCP used an address that you desire to assign to a particular host.
* ! If that is the case check files server(1,2,3), adjust them and run the playbook matching desired ip and DHCP address first.
* In my case DHCP gave me address 192.168.0.31 on server3. I wanted to use this address for server1, therefor I had to run the playbook for server3 first in order to unlock the address availability.

### Run the playbooks to change the DHCP to Static

```html
ansible-playbook pi_files/netplan/pi-static-playbook-server3.yaml -i git-prep 
ansible-playbook pi_files/netplan/pi-static-playbook-server2.yaml -i git-prep 
ansible-playbook pi_files/netplan/pi-static-playbook-server1.yaml -i git-prep 
```

* After applying the changes, substitute the old IPs with the new IPs in the Inventory git-prep.

### Test connectivity of you hosts use ansible

 Test your connection to pis ssh with the changed IP. You know what are you looking for already.

```sh
ansible all -m ping -i git-prep
```

### Modify nodes cgroups

Go to ```/boot/firmware``` and do

```sh
sudo vim cmdline.txt
```

append the line after ```quiet splash``` with ```swapaccount=1 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory systemd.unified_cgroup_hierarchy=0```.

## k8s_cluster_setup

Run playbook kubeadm.yaml

```sh
ansible-playbook playbooks/kubeadm.yml -i git-prep
```

* Important! If you run the playbook with "--check" it will fail in stage Containerd, becaus ```/etc/containerd/config.toml``` does not yet exist. In order to run a successful dry-run you will need to create this dir and file for each node.

## Choose your master-node

`Before you begin`

* If you are, consider using bind9 and create dns server for your k8s --control-plane-endpoint=server1.local.
[More info](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init)
* Add this flag with the IP or dns name if you are ever to add more than one master-node.
Select your master node and run

```sh
sudo kubeadm init
```

### Join the worker nodes by using the token form the master-node

You will get a token for the worker-nodes
Run that token as root on you worker-nodes so they can join the cluster.

```sh
sudo -s
```

```kubeadm join example.com:6443 --token vt4ua6.wcma2y8pl4menxh2    --discovery-token-ca-cert-hash sha256:<Token_hash>```

### Add your user to the group

Run the following so you don't have to sudo all the time.

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Test local network and node readiness

Run the following test.  

```sh
kubectl cluster-info
kubectl get nodes
```

You nodes are not ready because you don't have container network interface within yet.

### Install Calico Network Add-on

Install Calico Network Add-on (becaus it works right out of the box, however you are welcome to select other network addons if you wish).

```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml
```

### Final checks and tests

Check the system namespace for calico/network pods ```$ kubectl get pods -n kube-system```.

Run the following test. You nodes are ready.

```sh
kubectl cluster-info
kubectl get nodes
```

### Conclusion

Prepping the cluster-nodes takes 3 times longer then installing the cluster.
Happy Navigating!

### Considerations

* This setup is not secure and it is meant to run on private networks, for practicing only!
* Port restrictions can be done with IP tables and NF tables to mach security posture.
* You will also need to edit the kubeadm.yaml playbook as the checks are currently commented out.
[more](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)

### Next topic

* nfs-server [Setup NFS server for persistent volume storage](https://github.com/emilpeychev/nfs-server)
