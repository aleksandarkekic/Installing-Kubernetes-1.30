
# ➖➖➖➖➖➖  Installing-Kubernetes-1.30 ➖➖➖➖➖➖
# Configure the control and worker nodes
For each of the nodes (control and workers), you will need to install the necessary packages and configure the system. Each node needs to have containerd, crictl, kubeadm, kubelet, and kubectl installed.

Start by SSH’ing into each node and make sure you are running as root.
```bash
sudo -i
```
---
# System Updates
```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
Update the system and install necessary packages.
```bash
apt-get update && apt-get upgrade -y
```
# Install containerd
Kubernetes uses the Container Runtime Interface (CRI) to interact with container runtimes and containerd is the container runtime that Kubernetes uses (it was dockerd in the past).

We will need to install containerd on each node from the Docker repository and configure it to use systemd as the cgroup driver.
Add docker gpg key and repository.
```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Update packages and install containerd.
```bash
apt-get update
apt-get install containerd.io -y
```
Configure containerd to use systemd as the cgroup driver to use systemd cgroups.
```bash
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```
Update containerd to load the overlay and br_netfilter modules.
```bash
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
apt-get install containerd.io -y
```
Update kernel network settings to allow traffic to be forwarded.
```bash
cat << EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
Load the kernel modules and apply the sysctl settings to ensure they changes are used by the current system.
```bash
modprobe overlay
modprobe br_netfilter
sysctl --system
```
Verify containerd is running.
```bash
systemctl status containerd
```
# Install kubeadm, kubelet, and kubectl
The following tools are necessary for a successful Kubernetes installation:

kubeadm: the command to bootstrap the cluster.
kubelet: the component that runs on all of the machines in the cluster and does things like starting pods and containers.
kubectl: the command line tool to interact with the cluster.
Add the Kubernetes repository.
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
```
Update the system, install the Kubernetes packages, and lock the versions.
```bash
apt-get update
apt-get install -y kubelet=1.30.3-1.1 kubeadm=1.30.3-1.1 kubectl=1.30.3-1.1
apt-mark hold kubelet kubeadm kubectl
```
Enable kubelet.
```bash
systemctl enable --now kubelet
```

# Install critctl
CRI-O is an implementation of the Container Runtime Interface (CRI) used by the kubelet to interact with container runtimes.
Install crictl by downloading the binary to the system.
```bash
export CRICTL_VERSION="v1.30.1"
export CRICTL_ARCH=$(dpkg --print-architecture)
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$CRICTL_VERSION/crictl-$CRICTL_VERSION-linux-$CRICTL_ARCH.tar.gz
tar zxvf crictl-$CRICTL_VERSION-linux-$CRICTL_ARCH.tar.gz -C /usr/local/bin
rm -f crictl-$CRICTL_VERSION-linux-$CRICTL_ARCH.tar.gz
```
Verify crictl is installed.
```bash
crictl version
```
# Install kubernetes on control node
The control node is where the Kubernetes control plane components will be installed. This includes the API server, controller manager, scheduler, and etcd. The control node will also run the CNI plugin to provide networking for the cluster.

# Control plane installation with kubeadm
Using kubeadm, install the Kubernetes with the kubeadm init command. This will install the control plane components and create the necessary configuration files.
```bash
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs
```
