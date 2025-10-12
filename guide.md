# **Prerequisites:**

There are **two server types** used in deployment of Kubernetes clusters:

- **Master**: A Kubernetes Master **(control plane)**
- **Node**: A Node is a system that provides the run-time environments for the containers

**The minimum requirements for a viable setup are**

- **Memory**: 2 GB or more of RAM per machine • **CPUs**: At least 2 CPUs on the control plane machine. • **Internet connectivity** for pulling containers required (Private registry can also be used) • **Full network connectivity(using bridged network) between machines in the cluster**

**ssh port:** fix the ssh port to range of 50000 to make sure the security and then open the firewall for it to connect from your local machine

set ip for each vms

```bash
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.0.201/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8,8.8.4.4]
```

name server 

```bash
hostnamectl set-hostname [your-master-server-name]
```

hostname

```bash
petops@master:~$ cat /etc/host
192.168.0.201 master
192.168.0.202 worker1
192.168.0.203 worker2
```

**add the whitelist:** as my habit to improve the security i add the same network to the whitelist in case you want to specify an IP instead of all the internal network you can set it like this syntax: `sudo ufw allow from 192.168.0.223 to any port 6443 proto tcp`

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.0/24
```

# The main step

1. Create a new user for 3 servers

```bash
 adduser petops
 usermod -aG sudo petops
 newgrp petops
```

1. Disable Swap

```bash
sudo swapoff -a
```

We need to 標注 the final line in /etc/fstab by going it and 標注
or we can use below syntax for fast

```bash
sudo sed -i '/swap/ s/^/#/' /etc/fstab
```

Results:

```bash
#/swap.img      none    swap    sw      0       0
```

1. preparing Linux system to handle container networking properly before installing containerd

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

-> This step opens up the Linux system to allow Kubernetes networking to work. It lets pods talk to each other, and lets kube-proxy control traffic safely.

1. install container runtime

There are 3 container runtimes: containerd, docker, and cri

Historically, Kubernetes used Docker as its container runtime.
But since Kubernetes v1.24, Docker support was removed and replaced by containerd (and CRI-O).
We use containerd in this case instead of Docker or cri

# Add repo and Install packages

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

```

# Install containerd

```bash
sudo apt update -y
sudo apt install -y [containerd.io](http://containerd.io/)
```

# Config containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

1. Install kubeadm, kubelet, kubectl

I got the problem in here when installing the keyring of k8s with v1.30. 

```bash
W: OpenPGP signature verification failed: https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.30/deb  InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 234654DA9A296436
E: The repository 'https://pkgs.k8s.io/core:/stable:/v1.30/deb  InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
N: Some sources can be modernized. Run 'apt modernize-sources' to do so.
```

Previously, the keyring file type was unsupported. So I have to fix it by correctly adding the Kubernetes signing key using `Release.key` and storing it as a `.gpg` file under `/etc/apt/keyrings`.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
```

we need to hold the recent version of kubelet kubeadm kubectl cause when we use the update syntax some components will be updated leading to our k8s cluster getting errors

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

1. Initialize the cluster on the master node first

use this below syntax for your master node

```bash
sudo kubeadm init
```

K8s will tell you where the syntax will be used:

```bash
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.201:6443 --token gpthzo.a03lztlp91fxon5n \
        --discovery-token-ca-cert-hash sha256:c59fc8bae0f60edf2e02cb60f2028c0ed9542320916c6395fa4a22cf7ec89eaa
```

Use these in your master node 

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

And copy these syntaxs to your work nodes to tell k8s that these vms are worker nodes:

```bash
sudo kubeadm join 192.168.0.201:6443 --token gpthzo.a03lztlp91fxon5n \
        --discovery-token-ca-cert-hash sha256:c59fc8bae0f60edf2e02cb60f2028c0ed9542320916c6395fa4a22cf7ec89eaa
```

1. Add CNI

Checking the node status, they are not ready

```bash
petops@master:~$ kubectl get nodes
NAME      STATUS     ROLES           AGE     VERSION
master    NotReady   control-plane   6m51s   v1.30.14
worker1   NotReady   <none>          44s     v1.30.14
worker2   NotReady   <none>          37s     v1.30.14
```

when we check the `kubectl describe nodes worker1` we can know that the CNI plugin is not configured 

```bash
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 12 Oct 2025 17:40:10 +0000   Sun, 12 Oct 2025 17:39:39 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 12 Oct 2025 17:40:10 +0000   Sun, 12 Oct 2025 17:39:39 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 12 Oct 2025 17:40:10 +0000   Sun, 12 Oct 2025 17:39:39 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Sun, 12 Oct 2025 17:40:10 +0000   Sun, 12 Oct 2025 17:39:39 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

# Installing CNI for k8s cluster

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/calico.yaml
```

Checking the status again

```bash
petops@master:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
master    Ready    control-plane   18m   v1.30.14
worker1   Ready    <none>          12m   v1.30.14
worker2   Ready    <none>          12m   v1.30.14
```
