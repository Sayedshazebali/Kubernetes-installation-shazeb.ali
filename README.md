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

**Enable iptables Bridged Traffic on all the Nodes**
For ALL THE NODES
IPtables to see bridged traffic. Here we are tweaking some kernel parameters and setting them using sysctl


'''
`cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF`
'''

# Apply sysctl params without reboot
sudo sysctl --system
