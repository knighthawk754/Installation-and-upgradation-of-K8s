# Installation and Upgradation of Kubernates on RHEL
## Prerequisites
- **Admin rights:** Sudo user
- **Control Plane Node:** Minimum 2 CPU cores and 2GB of RAM (4GB+ is recommended).
- **Worker Nodes:** At least 1 CPU core and 1GB of RAM (2GB+ is recommended).
- **Operating System:** A compatible Linux distribution (e.g., Rocky Linux 9, RHEL 9, CentOS 9, Ubuntu 20.04+, Debian 10+, etc.).

The control plane node should have a unique name. (e.g., k8s-master01)
```$ sudo hostnamectl set-hostname "k8s-master01" && exec bash```
Workers node should have a numerical order (e.g., k8s-worker01..02..03)
``` $ sudo hostnamectl set-hostname "k8s-worker01" && exec bash```
Add the hosts on each node in /etc/hosts. 
### Master node
![alt text](image-1.png)

### Worker node 
![alt text](image-2.png)

### Disable Swap Space on Each Node

> This activity should be on both worker node and Master node

```
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

```
### Adjust SELinux and Firewall Rules

```
$ sudo setenforce 0
$ sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/sysconfig/selinux
```
### Add Kernel Modules and Parameters

```
sudo echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/containerd.conf && sudo modprobe overlay && sudo modprobe br_netfilter
```
Kubernetes requires specific kernel parameters for effective network traffic management:

1. **net.bridge.bridge-nf-call-iptables** = 1 enables bridged traffic to pass through iptables, crucial for routing between nodes and pods.
2. **net.ipv4.ip_forward** = 1 allows IP forwarding for pod-to-pod communication across node interfaces.
3. **net.bridge.bridge-nf-call-ip6tables** = 1 manages IPv6 traffic on bridged interfaces through ip6tables, important for IPv6 networking environments.

``` 
sudo echo -e "net.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\nnet.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/k8s.conf
```
Then add these parameters by running these command:
``` $ sudo sysctl --system ```

### Install Containerd Runtime
```
$ sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
Now, run following dnf command to install containerd on all the nodes.
``` sudo dnf install containerd.io -y ```

Configure containerd so that it will use systemdcgroup, execute the following commands on each node.
```
$ containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
$ sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
Restart and enable containerd service using beneath commands

```
$ sudo systemctl restart containerd
$ sudo systemctl enable containerd
```
Verify containerd service status, run

``` $ sudo systemctl status containerd ```

### Install Kubernetes tools

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
Next, install Kubernetes tools by running following dnf command,
``` sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes ```
After installing Kubernetes tools, start the kubelet service on each node.
``` $ sudo systemctl enable --now kubelet  ```

### Step: Install Kubernetes Cluster
> Now, we are all set to install Kubernetes cluster. Run beneath Kubeadm command to initialize the Kubernetes cluster from the master node.

``` sudo kubeadm init --control-plane-endpoint=k8s-master01 ```

Once above command is executed successfully, we will get following output,
![alt text](image-3.png)

From the output above make a note of the command which will be executed on the worker nodes to join the Kubernetes cluster.

To start interacting with Kubernetes cluster, run the following commands on the master node.

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
***Next, join the worker nodes to the cluster, run following Kubeadm command from the worker nodes.***

# Upgradation of Kubernates

### Install Kubernetes tools
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
## Upgrade Steps
1. **Upgrade Control Plane Nodes**

**Update kubeadm:**

```sudo yum install -y kubeadm --disableexcludes=kubernetes```

2. **Plan the Upgrade:**
   
``` kubeadm upgrade plan ```

3. **Apply the Upgrade:**
``` sudo kubeadm upgrade apply v1.29.x ```

> Replace v1.29.x with the exact version you want to install.

4. **Update kubelet and kubectl:**
```
sudo yum install -y kubelet kubectl --disableexcludes=kubernetes
sudo systemctl restart kubelet
```
5. **Verify Control Plane:**
Confirm the control plane is running the new version:

```
kubectl get nodes
kubectl version
```
## Upgrade Worker Nodes

  - **Drain the Node**
     On the control plane, drain the worker node
    
      ``` kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data ```
    
  - **Upgrade kubeadm on the Node:**
    
    ``` sudo yum install -y kubeadm --disableexcludes=kubernetes ```
    
  - **Update Configuration:**
    
   ``` sudo kubeadm upgrade node ```
   
 - **Update kubelet and kubectl:**
   
  ```
  sudo yum install -y kubelet kubectl --disableexcludes=kubernetes
  sudo systemctl restart kubelet
  ```

- **Uncordon the Node:**
  
  On the control plane, uncordon the node:

  ```
    kubectl uncordon <node-name>
  ```
- **Verify Node:**
  
``` kubectl get nodes ```

