# K8s
# Author: Milen Mladenov
# This document is created with goal to present deploying 3 node k8s cluster - 1 master node and 2 worker nodes with kubeadm
# Container runtime which is used in this project is containerd, instead of most commonly used docker.

# Project is started on 16.11.22, currently it is not finished
# Currently, in this document, available information is only for deploying k8s master node, details for configuring worker nodes should be added soon.
# Also document is not formated
# Idea for this project can be extended and may include additional features


# Prerequisites - provisioning lab enviorment.
#Provisioning 3 VM's on virtualbox, using Ubuntu server 22.04 LTS - image from osboxes.org
#Master node - 2GB, 2 CPU - k8sMaster
#2 x worker nodes - 1GB, 1 CPU - k8sWorker, k8sWorker2

# Installing starting and enabling SSH daemon
# In case that ssh is not installed run following commands
# It should be installed and enabled on VM's, if it's installed or you prefer to work directly on VM terminal this step can be skipped.
'sudo apt update'
'sudo apt install ssh'
'sudo systemctl start sshd'
'sudo systemctl enable sshd'

#With goal of efficiency we will configure passwordless authentication between local PC on which VMs are installed and VM's
#We will use command 'ssh-keygen' to generate ssh key pair named "k8s", we won't use passphrase, because it is just for testing purposes.
#In production enviorment it's not acceptable, also in production enviorment is recomanded to generate different key-pair for every node.
#Using following command:

'ssh-copy-id -i k8s.pub osboxes@192.168.100.92'

#IP address must be on targeted node
#Public key will be copied to targeted node(Operation must be performed on all nodes).
#After successfull authentication we can simply connect to nodes trough following command:
'ssh -i k8s osboxes@192.168.100.92'
#IP address of targeted node

# Currently firewall is not installed on our enviorment
# In case that firewall is installed and enabled you should disabled it with following command:
'sudo ufw disable'
# In case that firewall service is necessary (e.x. in production) you must follow kubernetes requirements for port configuration descried in official documentation.
# https://kubernetes.io/docs/reference/ports-and-protocols/

#You MUST disable swap in order for the Kubelet to work properly.
# First disable swap with following command:
'sudo swapoff -a'
# And then disable swap on startup in /etc/fstab
'sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab'

# Curl utility should be installed with following command
# If you have curl already installed this step should be skipped
'sudo apt install curl'


## Configuring containerd as container runtime for Kubernetes

#First load two modules in the current running environment and configure them to load on boot

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

#Configure required sysctl to persist across system reboots

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables=1
EOF

# Apply sysctl parameters without reboot to current running enviroment

sudo sysctl --system

# Installing containerd packages

'sudo apt-get update'
'sudo apt-get install -y containerd'

# Create a containerd configuration file

'sudo mkdir -p /etc/containerd'
'sudo containerd config default | sudo tee /etc/containerd/config.toml'

# Set the cgroup driver for runc to systemd
# In /etc/containerd/config.toml change the value for SystemCgroup from false to true.
# SystemdCgroup = true

# And finaly restart containerd to apply configuration
'sudo systemctl restart containerd'


# Now we should install finally install k8s with kubeadm
# Same steps are described in k8s documentation https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

# Update the apt package index and install packages needed to use the Kubernetes apt repository:

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Download the Google Cloud public signing key
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

# Add the Kubernetes apt repository:
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# To initialize the control-plane/master node run:

sudo kubeadm init

# After completion of previous command make sure that you execute following commands:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

