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
sudo modprobe br_netfilter`

*sysctl params required by setup, params persist across reboots*

`cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
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

``OS="xUbuntu_22.04"

VERSION="1.28"

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF``


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

+ # We can use the following commands to find the latest versions.

`sudo apt update
apt-cache madison kubeadm | tac`

**Specify the version as shown below.**

`sudo apt-get install -y kubelet=1.28.2-00 kubectl=1.28.2-00 kubeadm=1.28.2-00`

**Or, to install the latest version from the repo use the following command without specifying any version.**

`sudo apt-get install -y kubelet kubeadm kubectl`

**Add hold to the packages to prevent upgrades.**

`sudo apt-mark hold kubelet kubeadm kubectl`

Now we have all the required utilities and tools for configuring Kubernetes components using kubeadm.

**Add the node IP to KUBELET_EXTRA_ARGS.**

`sudo apt-get install -y jq
local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF`

# Initialize Kubeadm On Master Node To Setup Control Plane

Here we need to consider two options.

**Master Node with Private IP:** If you have nodes with only private IP addresses the API server would be accessed over the private IP of the master node.
**Master Node With Public IP**: If you are setting up a Kubeadm cluster on Cloud platforms and you need master Api server access over the Public IP of the master node server.

**Only the Kubeadm initialization command differs for Public and Private IPs.**

Execute the commands in this section only on the master node.

If we are using a **Private IP** for the master Node,

Set the following environment variables. **Replace 10.0.0.10 ** with the IP of our master node.

`IPADDR="10.0.0.10"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"`

If we want to use the **Public IP** of the master node,

Set the following environment variables. The **IPADDR variable** will be automatically set to the server’s public IP using ifconfig.me curl call. You can also replace it with a public IP address

`IPADDR=$(curl ifconfig.me && echo "")
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"`

**Now, initialize the master node control plane configurations using the kubeadm command.**

For a **Private IP address-based setup** use the following init command.

`sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap`

--ignore-preflight-errors Swap is actually not required as we disabled the swap initially.

**For Public IP address-based setup use the following init command.**

Here instead of --apiserver-advertise-address we use --control-plane-endpoint parameter for the API server endpoint.

`sudo kubeadm init --control-plane-endpoint=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap`

All the other steps are the same as configuring the master node with private IP.

On a successful kubeadm initialization, you should get an output with kubeconfig file location and the join command with the token as shown below. Copy that and save it to the file. we will need it for joining the worker node to the master.


![image](https://github.com/Sayedshazebali/Kubernetes-installation-shazeb.ali/assets/115386350/8454e478-c422-426d-a41f-7fc9e86efe41)

Use the following commands from the output to create the kubeconfig in master so that you can use kubectl to interact with cluster API.

`mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config`

Now, verify the kubeconfig by executing the following kubectl command to list all the pods in the kube-system namespace.

`kubectl get po -n kube-system`

**You verify all the cluster component health statuses using the following command.**

`kubectl get --raw='/readyz?verbose'`

![image](https://github.com/Sayedshazebali/Kubernetes-installation-shazeb.ali/assets/115386350/ea38d311-414a-460e-8a88-1721f35e8da2)

# Install Calico Network Plugin for Pod Networking

Kubeadm does not configure any network plugin. You need to install a network plugin of your choice for Kubernetes pod networking and enable network policy.

**I am using the Calico network plugin for this setup.**

`kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O

kubectl create -f custom-resources.yaml`

# Join Worker Nodes To Kubernetes Master Node

We have set up cri-o, kubelet, and kubeadm utilities on the worker nodes as well.

Now, let’s join the worker node to the master node using the Kubeadm join command you have got in the output while setting up the master node.

**If we missed copying the join command**, execute the following command in the master node to recreate the token with the join command.

`kubeadm token create --print-join-command`

Here is what the command looks like. Use sudo if you running as a normal user. This command performs the TLS bootstrapping for the nodes.

`sudo kubeadm join 10.128.0.37:6443 --token j4eice.33vgvgyf5cxw4u8i \
    --discovery-token-ca-cert-hash sha256:37f94469b58bcc8f26a4aa44441fb17196a585b37288f85e22475b00c36f1c61`

    
On successful execution, you will see the output saying, “This node has joined the cluster”.

![image](https://github.com/Sayedshazebali/Kubernetes-installation-shazeb.ali/assets/115386350/885e743a-70b7-442a-bcdc-8dd8af3caa3a)

Now execute the** kubectl command from the master node to check if the node is added to the master.**

`kubectl get nodes`


