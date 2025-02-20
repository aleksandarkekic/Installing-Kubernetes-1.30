
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
