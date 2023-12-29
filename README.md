**INSTALLING KUBERNETES CLUSTER USING KUBEADM**
![image](https://github.com/Sayedshazebali/Kubernetes-installation-shazeb.ali/assets/115386350/eba01339-7c67-4785-9da8-12b95d217ca0)

In this Image its show how Kubeadm init work when we install with the help of kubeadm tool

**KUBEADM SETUP PREREQUISITES**
+ Minimum 2 Ubuntu nodes 1for master and one for worker node we can use more than that also but i am using 2 instance
+ The master node should be min 2 vCpu and 2gb Ram
+ For worker a min of 1vCpu and 2 gb Ram

**Kubeadm port requirements**
*For Control-plane node*
+ 6443 : for kube-api-server it used by all
+ 2379-2380: for etcd-server client Api it used by kube-api-server and etcd
+ 10250: for kubelet Api for self and control plan
+ 10251: Kube-scheduler for self
+ 10252: kubecontroller-manager for self
*Wroker Node
+ 10250: Kubelet API for self and control plane
+ 30000-32767: for nodeport-service: for all

+ **Enable iptables Bridged Traffic on all the Nodes**
For ALL THE NODES
IPtables to see bridged traffic. Here we are tweaking some kernel parameters and setting them using sysctl


`cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

*sysctl params required by setup, params persist across reboots*
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF`

+ **Apply sysctl params without reboot**
`sudo sysctl --system`

+ **Disable swap on all the Nodes**
For kubeadm to work properly, we need to disable swap on all the nodes using the following command.
`sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true`
The fstab entry will make sure the swap is off on system reboots.

we can also, control swap errors using the kubeadm parameter --ignore-preflight-errors Swap.


# Install CRI-O Runtime On All The Nodes
The basic requirement for a Kubernetes cluster is a container runtime. we can have any one of the following container runtimes.
+ CRI-O
+ containerd
+ Docker Engine (using cri-dockerd)
  
We will be using CRI-O instead of Docker for this setup as Kubernetes deprecated Docker engine

As a first step, we need to install cri-o on all the nodes. Execute the following commands on all the nodes.
Enable cri-o repositories for version 1.28

`OS="xUbuntu_22.04"

VERSION="1.28"

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF`


+ **Add the GPG keys for CRI-O to the system’s list of trusted keys.**
`curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -`

+ **Update and install crio and crio-tools.**
`sudo apt-get update
sudo apt-get install cri-o cri-o-runc cri-tools -y`

+ **Reload the systemd configurations and enable cri-o.**
  
`sudo systemctl daemon-reload
sudo systemctl enable crio --now`

_Note: The cri-tools contain crictl, a CLI utility to interact with the containers created by the container runtime. When you use container runtimes other than Docker, you can use the crictl utility to debug containers on the nodes_

+ **Install Kubeadm & Kubelet & Kubectl on all Nodes**
                     
Install the required dependencies.
`sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl`

+ **Download the GPG key for the Kubernetes APT repository.**
  
`sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://dl.k8s.io/apt/doc/apt-key.gpg`

+ **Add the Kubernetes APT repository to your system.**
`echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list`

+ **Update apt repo**
`sudo apt-get update -y`




