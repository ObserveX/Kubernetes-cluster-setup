# Kubernetes-cluster-setup

KUBERNETES 1.26
CONTAINERD 1.6.16
UBUNTU 22.04

## On Master and Worker Nodes: 

```
sudo -s
```
Appends Master and Worker Nodes IP addresses and corresponding hostnames to the /etc/hosts file
```
printf "\n192.168.15.93 k8s-control\n192.168.15.94 k8s-2\n\n" >> /etc/hosts
```
Appends two kernel modules to be loaded at boot time
```
printf "overlay\nbr_netfilter\n" >> /etc/modules-load.d/containerd.conf
```
Configures kernel parameters to allow for container networking
```
printf "net.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\n" >> /etc/sysctl.d/99-kubernetes-cri.conf
```
Reloads the sysctl configuration
```
sysctl --system
```
Downloads and installs the container runtime and a systemd service file for containerd
```
wget https://github.com/containerd/containerd/releases/download/v1.6.16/containerd-1.6.16-linux-amd64.tar.gz -P /tmp/
tar Cxzvf /usr/local /tmp/containerd-1.6.16-linux-amd64.tar.gz
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now containerd
```
Downloads and installs runc, the CLI tool for running containers
```
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64 -P /tmp/
install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc
```
Downloads and installs the CNI plugins, which are necessary for configuring container networking
```
wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz -P /tmp/
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin /tmp/cni-plugins-linux-amd64-v1.2.0.tgz
```
Create Containerd's configuration file
```
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml   <<<<<<<<<<< manually edit and change systemdCgroup to true
systemctl restart containerd
```
Disables swap
```
swapoff -a  <<<<<<<< just disable it in /etc/fstab instead
```
Update and install the necessary packages.
```
apt-get update
apt-get install -y apt-transport-https ca-certificates curl
```
Adds the Kubernetes repository key and adds the Kubernetes repository to the apt sources list
```
curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
```
Update apt sources and reboot the server
```
apt-get update
reboot
```
Install Kubernetes components
```
apt-get install -y kubelet=1.26.1-00 kubeadm=1.26.1-00 kubectl=1.26.1-00
apt-mark hold kubelet kubeadm kubectl
```

# check swap config, ensure swap is 0
```
free -m
```

## ONLY ON CONTROL NODE
Initialize a Kubernetes control plane
```
kubeadm init --pod-network-cidr 10.10.0.0/16 --kubernetes-version 1.26.1 --node-name k8s-control
```


Add Calico 3.25 CNI 
```
*** https://docs.tigera.io/calico/3.25/getting-started/kubernetes/quickstart
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml
vi custom-resources.yaml <<<<<< edit the CIDR for pods if its custom
kubectl apply -f custom-resources.yaml
```
# get worker node commands to run to join additional nodes into cluster
```
kubeadm token create --print-join-command
```

## ONLY ON WORKER nodes
```
Run the command from the token create output above on all the worker nodes
```

To Verify, Run this command on master node
```
kubectl get nodes
```
