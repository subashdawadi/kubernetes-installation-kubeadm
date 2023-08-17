# kubernetes-installation-kubeadm# kubernetes cluster setup using containerd for production build with binary

Here is step by step approach to build kubernetes cluster using containerd as container runtime from binary.here i am using three ubuntu 22.04 machine having 2gb ram and 2 cpu . one is master node and two worker node. 

## 1. For all three machine
## Set hostname for three machine


first set hostname for three nodes run as root in three machine.

    hostnamectl set-hostname master
    hostnamectl set-hostname worker01
    hostnamectl set-hostname worker02

## Disable swap

Disable swap

    swapoff -a
    sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab


## Install Containerd

Install containerd as container runtime.

    wget https://github.com/containerd/containerd/releases/download/v1.7.3/containerd-1.7.3-linux-amd64.tar.gz
Unzip to /usr/local

    tar Cxzvf /usr/local containerd-1.7.3-linux-amd64.tar.gz
    
Download service file and make service file.

    wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
    mkdir -p /usr/local/lib/systemd/system
    mv containerd.service /usr/local/lib/systemd/system/containerd.service
reload and enable

    systemctl daemon-reload
    systemctl enable --now containerd

## Install Runc

install runc

    wget https://github.com/opencontainers/runc/releases/download/v1.1.9/runc.amd64
    install -m 755 runc.amd64 usr/local/sbin/runc
    

## Install CNI

install cni

    wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
    mkdir -p /opt/cni/bin
    tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz


## Install CRICTL

    VERSION="v1.26.0"
    wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
    tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
    rm -f crictl-$VERSION-linux-amd64.tar.gz
    cat <<EOF | sudo tee /etc/crictl.yaml
    runtime-endpoint: unix:///run/containerd/containerd.sock
    image-endpoint: unix:///run/containerd/containerd.sock
    timeout: 2
    debug: false
    pull-image-on-create: false
    EOF

## Forwarding IPv4 and letting iptables see bridged traffic

ip forwarding

    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    sudo modprobe overlay
    sudo modprobe br_netfilter
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward = 1
    EOF
    sudo sysctl --system

## Install kubelet kubeadm kubectl

    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
Download the Google Cloud public signing key:

    curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
    
Add the Kubernetes apt repository:

    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl



## Run on Master Node and follow the instructions

    kubeadm config images pull
    kubeadm init --pod-network-cidr=10.244.0.0/16
    
    
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    

## Install weavenet plugin on master as normal user

refer  https://www.weave.works/docs/net/latest/kubernetes/kube-addon/

    kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Join in worker nodes
weget this command after we initilized cluster in master node. below is example command.

    sudo kubeadm join <your control node ip>:6443 --token <yourtoken>
    --discovery-token-ca-cert-hash
    <your SHA>


## Test in master node.

    kubectl get nodes

# enjoy!
